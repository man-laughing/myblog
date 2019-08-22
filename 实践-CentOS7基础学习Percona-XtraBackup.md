
---
title: 实践-CentOS7基础学习Percona-XtraBackup
date: 2019/08/22
tags: 
   mysql
---


### 环境
|OS          |Kernel   |   IP   |Mysql version|Percona-XtraBackup|
|------------|--------:|:-------|:------------|:---|
|CentOS 7.6  |3.10.0   |10.99.66.154|5.6.45   |2.4.15|
 

### 先决条件

* 保证你的机器可以连通网络
* 保证你的数据库服务运行正常（安装过程略）
 
### 安装

* 下载安装包
参考下载地址：https://www.percona.com/downloads/Percona-XtraBackup-2.4/LATEST/

```bash
[root@localhost ~]# cd /usr/local/src
[root@localhost src]# wget -S "https://www.percona.com/downloads/Percona-XtraBackup-2.4/Percona-XtraBackup-2.4.15/binary/redhat/7/x86_64/Percona-XtraBackup-2.4.15-r544842a-el7-x86_64-bundle.tar"
[root@localhost src]#
[root@localhost src]# tar xf Percona-XtraBackup-2.4.15-r544842a-el7-x86_64-bundle.tar
[root@localhost src]# ll *.rpm
-rw-rw-r-- 1 root root  7893808 Jul  5 16:00 percona-xtrabackup-24-2.4.15-1.el7.x86_64.rpm
-rw-rw-r-- 1 root root 39551472 Jul  5 16:00 percona-xtrabackup-24-debuginfo-2.4.15-1.el7.x86_64.rpm
-rw-rw-r-- 1 root root 13657136 Jul  5 16:00 percona-xtrabackup-test-24-2.4.15-1.el7.x86_64.rpm
[root@localhost src]#
[root@localhost src]# yum localinstall percona-xtrabackup-24-2.4.15-1.el7.x86_64.rpm percona-xtrabackup-24-debuginfo-2.4.15-1.el7.x86_64.rpm percona-xtrabackup-test-24-2.4.15-1.el7.x86_64.rpm
[root@localhost src]#
[root@localhost src]# rpm -ql percona-xtrabackup-24 |grep bin
/usr/bin/innobackupex
/usr/bin/xbcloud
/usr/bin/xbcloud_osenv
/usr/bin/xbcrypt
/usr/bin/xbstream
/usr/bin/xtrabackup

TIPS:
以上命令最主要的是 innobackupex 和 xtrabackup，前者是一个 perl 脚本，后者是 C/C++ 编译的二进制。

xtrabackup   是用来备份 InnoDB 表的，不能备份非 InnoDB 表，和 mysqld server 没有交互；
innobackupex 是用来备份非 InnoDB 表，同时会调用 xtrabackup 命令来备份 InnoDB 表，还会和 mysqld server 发送命令进行交互，如加读锁（FTWRL）、获取位点（SHOW SLAVE STATUS）等。简单来说，innobackupex 在 xtrabackup 之上做了一层封装。

一般情况下，我们是希望能备份 MyISAM 表的，虽然我们可能自己不用 MyISAM 表，但是 mysql 库下的系统表是 MyISAM 的，因此备份基本都通过 innobackupex 命令进行；另外一个原因是我们可能需要保存位点信息。

另外2个工具相对小众些，xbcrypt 是加解密用的；xbstream 类似于tar，是 Percona 自己实现的一种支持并发写的流文件格式。两都在备份和解压时都会用到（如果备份用了加密和并发）。
```


### 备份（物理备份）

* 备份本机的所有数据库

```bash
[root@localhost src]# xtrabackup -uroot -proot -S /usr/local/mysql/logs/mysql.sock --backup --target-dir=/data/mysqlbackup/
xtrabackup: recognized server arguments:
xtrabackup: recognized client arguments: --user=root --password=* --socket=/usr/local/mysql/logs/mysql.sock --backup=1 --target-dir=/data/mysqlbackup/
190822 17:24:33  version_check Connecting to MySQL server with DSN 'dbi:mysql:;mysql_read_default_group=xtrabackup;mysql_socket=/usr/local/mysql/logs/mysql.sock' as 'root'  (using password: YES).
190822 17:24:33  version_check Connected to MySQL server
190822 17:24:33  version_check Executing a version check against the server...
190822 17:24:33  version_check Done.
190822 17:24:33 Connecting to MySQL server host: localhost, user: root, password: set, port: not set, socket: /usr/local/mysql/logs/mysql.sock
Using server version 5.6.45-log
xtrabackup version 2.4.15 based on MySQL server 5.7.19 Linux (x86_64) (revision id: 544842a)
xtrabackup: uses posix_fadvise().
xtrabackup: cd to /usr/local/mysql/data/
xtrabackup: open files limit requested 0, set to 102400
xtrabackup: using the following InnoDB configuration:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = ./
xtrabackup:   innodb_log_files_in_group = 2
xtrabackup:   innodb_log_file_size = 50331648
InnoDB: Number of pools: 1
190822 17:24:33 >> log scanned up to (1633057)
xtrabackup: Generating a list of tablespaces
InnoDB: Allocated tablespace ID 1 for mysql/innodb_table_stats, old maximum was 0
190822 17:24:33 [01] Copying ./ibdata1 to /data/mysqlbackup/ibdata1
190822 17:24:33 [01]        ...done
190822 17:24:33 [01] Copying ./mysql/innodb_table_stats.ibd to /data/mysqlbackup/mysql/innodb_table_stats.ibd
190822 17:24:33 [01]        ...done
190822 17:24:33 [01] Copying ./mysql/innodb_index_stats.ibd to /data/mysqlbackup/mysql/innodb_index_stats.ibd
190822 17:24:33 [01]        ...done
190822 17:24:33 [01] Copying ./mysql/slave_relay_log_info.ibd to /data/mysqlbackup/mysql/slave_relay_log_info.ibd
190822 17:24:33 [01]        ...done
190822 17:24:33 [01] Copying ./mysql/slave_master_info.ibd to /data/mysqlbackup/mysql/slave_master_info.ibd
190822 17:24:33 [01]        ...done
190822 17:24:33 [01] Copying ./mysql/slave_worker_info.ibd to /data/mysqlbackup/mysql/slave_worker_info.ibd
190822 17:24:33 [01]        ...done
190822 17:24:33 [01] Copying ./ttt/t1.ibd to /data/mysqlbackup/ttt/t1.ibd
190822 17:24:33 [01]        ...done
190822 17:24:34 >> log scanned up to (1633057)
190822 17:24:34 Executing FLUSH NO_WRITE_TO_BINLOG TABLES...
190822 17:24:34 Executing FLUSH TABLES WITH READ LOCK...
190822 17:24:34 Starting to backup non-InnoDB tables and files
190822 17:24:34 [01] Copying ./mysql/db.frm to /data/mysqlbackup/mysql/db.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/db.MYI to /data/mysqlbackup/mysql/db.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/db.MYD to /data/mysqlbackup/mysql/db.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/user.frm to /data/mysqlbackup/mysql/user.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/user.MYI to /data/mysqlbackup/mysql/user.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/user.MYD to /data/mysqlbackup/mysql/user.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/func.frm to /data/mysqlbackup/mysql/func.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/func.MYI to /data/mysqlbackup/mysql/func.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/func.MYD to /data/mysqlbackup/mysql/func.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/plugin.frm to /data/mysqlbackup/mysql/plugin.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/plugin.MYI to /data/mysqlbackup/mysql/plugin.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/plugin.MYD to /data/mysqlbackup/mysql/plugin.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/servers.frm to /data/mysqlbackup/mysql/servers.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/servers.MYI to /data/mysqlbackup/mysql/servers.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/servers.MYD to /data/mysqlbackup/mysql/servers.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/tables_priv.frm to /data/mysqlbackup/mysql/tables_priv.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/tables_priv.MYI to /data/mysqlbackup/mysql/tables_priv.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/tables_priv.MYD to /data/mysqlbackup/mysql/tables_priv.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/columns_priv.frm to /data/mysqlbackup/mysql/columns_priv.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/columns_priv.MYI to /data/mysqlbackup/mysql/columns_priv.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/columns_priv.MYD to /data/mysqlbackup/mysql/columns_priv.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_topic.frm to /data/mysqlbackup/mysql/help_topic.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_topic.MYI to /data/mysqlbackup/mysql/help_topic.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_topic.MYD to /data/mysqlbackup/mysql/help_topic.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_category.frm to /data/mysqlbackup/mysql/help_category.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_category.MYI to /data/mysqlbackup/mysql/help_category.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_category.MYD to /data/mysqlbackup/mysql/help_category.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_relation.frm to /data/mysqlbackup/mysql/help_relation.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_relation.MYI to /data/mysqlbackup/mysql/help_relation.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_relation.MYD to /data/mysqlbackup/mysql/help_relation.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_keyword.frm to /data/mysqlbackup/mysql/help_keyword.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_keyword.MYI to /data/mysqlbackup/mysql/help_keyword.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/help_keyword.MYD to /data/mysqlbackup/mysql/help_keyword.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_name.frm to /data/mysqlbackup/mysql/time_zone_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_name.MYI to /data/mysqlbackup/mysql/time_zone_name.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_name.MYD to /data/mysqlbackup/mysql/time_zone_name.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone.frm to /data/mysqlbackup/mysql/time_zone.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone.MYI to /data/mysqlbackup/mysql/time_zone.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone.MYD to /data/mysqlbackup/mysql/time_zone.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_transition.frm to /data/mysqlbackup/mysql/time_zone_transition.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_transition.MYI to /data/mysqlbackup/mysql/time_zone_transition.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_transition.MYD to /data/mysqlbackup/mysql/time_zone_transition.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_transition_type.frm to /data/mysqlbackup/mysql/time_zone_transition_type.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_transition_type.MYI to /data/mysqlbackup/mysql/time_zone_transition_type.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_transition_type.MYD to /data/mysqlbackup/mysql/time_zone_transition_type.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_leap_second.frm to /data/mysqlbackup/mysql/time_zone_leap_second.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_leap_second.MYI to /data/mysqlbackup/mysql/time_zone_leap_second.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/time_zone_leap_second.MYD to /data/mysqlbackup/mysql/time_zone_leap_second.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/proc.frm to /data/mysqlbackup/mysql/proc.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/proc.MYI to /data/mysqlbackup/mysql/proc.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/proc.MYD to /data/mysqlbackup/mysql/proc.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/procs_priv.frm to /data/mysqlbackup/mysql/procs_priv.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/procs_priv.MYI to /data/mysqlbackup/mysql/procs_priv.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/procs_priv.MYD to /data/mysqlbackup/mysql/procs_priv.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/general_log.frm to /data/mysqlbackup/mysql/general_log.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/general_log.CSM to /data/mysqlbackup/mysql/general_log.CSM
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/general_log.CSV to /data/mysqlbackup/mysql/general_log.CSV
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/slow_log.frm to /data/mysqlbackup/mysql/slow_log.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/slow_log.CSM to /data/mysqlbackup/mysql/slow_log.CSM
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/slow_log.CSV to /data/mysqlbackup/mysql/slow_log.CSV
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/event.frm to /data/mysqlbackup/mysql/event.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/event.MYI to /data/mysqlbackup/mysql/event.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/event.MYD to /data/mysqlbackup/mysql/event.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/ndb_binlog_index.frm to /data/mysqlbackup/mysql/ndb_binlog_index.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/ndb_binlog_index.MYI to /data/mysqlbackup/mysql/ndb_binlog_index.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/ndb_binlog_index.MYD to /data/mysqlbackup/mysql/ndb_binlog_index.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/innodb_table_stats.frm to /data/mysqlbackup/mysql/innodb_table_stats.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/innodb_index_stats.frm to /data/mysqlbackup/mysql/innodb_index_stats.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/slave_relay_log_info.frm to /data/mysqlbackup/mysql/slave_relay_log_info.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/slave_master_info.frm to /data/mysqlbackup/mysql/slave_master_info.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/slave_worker_info.frm to /data/mysqlbackup/mysql/slave_worker_info.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/proxies_priv.frm to /data/mysqlbackup/mysql/proxies_priv.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/proxies_priv.MYI to /data/mysqlbackup/mysql/proxies_priv.MYI
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./mysql/proxies_priv.MYD to /data/mysqlbackup/mysql/proxies_priv.MYD
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/db.opt to /data/mysqlbackup/performance_schema/db.opt
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/cond_instances.frm to /data/mysqlbackup/performance_schema/cond_instances.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_waits_current.frm to /data/mysqlbackup/performance_schema/events_waits_current.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_waits_history.frm to /data/mysqlbackup/performance_schema/events_waits_history.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_waits_history_long.frm to /data/mysqlbackup/performance_schema/events_waits_history_long.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_waits_summary_by_instance.frm to /data/mysqlbackup/performance_schema/events_waits_summary_by_instance.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_waits_summary_by_host_by_event_name.frm to /data/mysqlbackup/performance_schema/events_waits_summary_by_host_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_waits_summary_by_user_by_event_name.frm to /data/mysqlbackup/performance_schema/events_waits_summary_by_user_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_waits_summary_by_account_by_event_name.frm to /data/mysqlbackup/performance_schema/events_waits_summary_by_account_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_waits_summary_by_thread_by_event_name.frm to /data/mysqlbackup/performance_schema/events_waits_summary_by_thread_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_waits_summary_global_by_event_name.frm to /data/mysqlbackup/performance_schema/events_waits_summary_global_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/file_instances.frm to /data/mysqlbackup/performance_schema/file_instances.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/file_summary_by_event_name.frm to /data/mysqlbackup/performance_schema/file_summary_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/file_summary_by_instance.frm to /data/mysqlbackup/performance_schema/file_summary_by_instance.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/socket_instances.frm to /data/mysqlbackup/performance_schema/socket_instances.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/socket_summary_by_instance.frm to /data/mysqlbackup/performance_schema/socket_summary_by_instance.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/socket_summary_by_event_name.frm to /data/mysqlbackup/performance_schema/socket_summary_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/host_cache.frm to /data/mysqlbackup/performance_schema/host_cache.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/mutex_instances.frm to /data/mysqlbackup/performance_schema/mutex_instances.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/objects_summary_global_by_type.frm to /data/mysqlbackup/performance_schema/objects_summary_global_by_type.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/performance_timers.frm to /data/mysqlbackup/performance_schema/performance_timers.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/rwlock_instances.frm to /data/mysqlbackup/performance_schema/rwlock_instances.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/setup_actors.frm to /data/mysqlbackup/performance_schema/setup_actors.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/setup_consumers.frm to /data/mysqlbackup/performance_schema/setup_consumers.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/setup_instruments.frm to /data/mysqlbackup/performance_schema/setup_instruments.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/setup_objects.frm to /data/mysqlbackup/performance_schema/setup_objects.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/setup_timers.frm to /data/mysqlbackup/performance_schema/setup_timers.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/table_io_waits_summary_by_index_usage.frm to /data/mysqlbackup/performance_schema/table_io_waits_summary_by_index_usage.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/table_io_waits_summary_by_table.frm to /data/mysqlbackup/performance_schema/table_io_waits_summary_by_table.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/table_lock_waits_summary_by_table.frm to /data/mysqlbackup/performance_schema/table_lock_waits_summary_by_table.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/threads.frm to /data/mysqlbackup/performance_schema/threads.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_stages_current.frm to /data/mysqlbackup/performance_schema/events_stages_current.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_stages_history.frm to /data/mysqlbackup/performance_schema/events_stages_history.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_stages_history_long.frm to /data/mysqlbackup/performance_schema/events_stages_history_long.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_stages_summary_by_thread_by_event_name.frm to /data/mysqlbackup/performance_schema/events_stages_summary_by_thread_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_stages_summary_by_host_by_event_name.frm to /data/mysqlbackup/performance_schema/events_stages_summary_by_host_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_stages_summary_by_user_by_event_name.frm to /data/mysqlbackup/performance_schema/events_stages_summary_by_user_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_stages_summary_by_account_by_event_name.frm to /data/mysqlbackup/performance_schema/events_stages_summary_by_account_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_stages_summary_global_by_event_name.frm to /data/mysqlbackup/performance_schema/events_stages_summary_global_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_statements_current.frm to /data/mysqlbackup/performance_schema/events_statements_current.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_statements_history.frm to /data/mysqlbackup/performance_schema/events_statements_history.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_statements_history_long.frm to /data/mysqlbackup/performance_schema/events_statements_history_long.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_statements_summary_by_thread_by_event_name.frm to /data/mysqlbackup/performance_schema/events_statements_summary_by_thread_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_statements_summary_by_host_by_event_name.frm to /data/mysqlbackup/performance_schema/events_statements_summary_by_host_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_statements_summary_by_user_by_event_name.frm to /data/mysqlbackup/performance_schema/events_statements_summary_by_user_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_statements_summary_by_account_by_event_name.frm to /data/mysqlbackup/performance_schema/events_statements_summary_by_account_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_statements_summary_global_by_event_name.frm to /data/mysqlbackup/performance_schema/events_statements_summary_global_by_event_name.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/hosts.frm to /data/mysqlbackup/performance_schema/hosts.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/users.frm to /data/mysqlbackup/performance_schema/users.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/accounts.frm to /data/mysqlbackup/performance_schema/accounts.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/events_statements_summary_by_digest.frm to /data/mysqlbackup/performance_schema/events_statements_summary_by_digest.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/session_connect_attrs.frm to /data/mysqlbackup/performance_schema/session_connect_attrs.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./performance_schema/session_account_connect_attrs.frm to /data/mysqlbackup/performance_schema/session_account_connect_attrs.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./ttt/db.opt to /data/mysqlbackup/ttt/db.opt
190822 17:24:34 [01]        ...done
190822 17:24:34 [01] Copying ./ttt/t1.frm to /data/mysqlbackup/ttt/t1.frm
190822 17:24:34 [01]        ...done
190822 17:24:34 Finished backing up non-InnoDB tables and files
190822 17:24:34 [00] Writing /data/mysqlbackup/xtrabackup_binlog_info
190822 17:24:34 [00]        ...done
190822 17:24:34 Executing FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS...
xtrabackup: The latest check point (for incremental): '1633057'
xtrabackup: Stopping log copying thread.
.190822 17:24:34 >> log scanned up to (1633057)

190822 17:24:34 Executing UNLOCK TABLES
190822 17:24:34 All tables unlocked
190822 17:24:34 Backup created in directory '/data/mysqlbackup/'
MySQL binlog position: filename 'mysql-bin.000003', position '4322', GTID of the last change '3b827fcb-bf40-11e9-81f9-fa163ec8903f:1-13'
190822 17:24:34 [00] Writing /data/mysqlbackup/backup-my.cnf
190822 17:24:34 [00]        ...done
190822 17:24:34 [00] Writing /data/mysqlbackup/xtrabackup_info
190822 17:24:34 [00]        ...done
xtrabackup: Transaction log of lsn (1633057) to (1633057) was copied.
190822 17:24:35 completed OK!
```

* 查看备份的文件

```bash
[root@localhost mysql]#  ls -l /data/mysqlbackup/
total 12316
-rw-r----- 1 root root      481 Aug 22 17:24 backup-my.cnf
-rw-r----- 1 root root 12582912 Aug 22 17:24 ibdata1
drwxr-x--- 2 root root     4096 Aug 22 17:24 mysql
drwxr-x--- 2 root root     4096 Aug 22 17:24 performance_schema
drwxr-x--- 2 root root       48 Aug 22 17:24 ttt
-rw-r----- 1 root root       64 Aug 22 17:24 xtrabackup_binlog_info
-rw-r----- 1 root root      135 Aug 22 17:24 xtrabackup_checkpoints
-rw-r----- 1 root root      589 Aug 22 17:24 xtrabackup_info
-rw-r----- 1 root root     2560 Aug 22 17:24 xtrabackup_logfile
```


### 恢复（物理恢复）

* 准备阶段

```bash
[root@localhost mysql]# 
[root@localhost mysql]# systemctl stop mysqld
[root@localhost mysql]# xtrabackup --prepare --target-dir=/data/mysqlbackup/
xtrabackup: recognized server arguments: --innodb_checksum_algorithm=innodb --innodb_log_checksum_algorithm=innodb --innodb_data_file_path=ibdata1:12M:autoextend --innodb_log_files_in_group=2 --innodb_log_file_size=50331648 --innodb_fast_checksum=0 --innodb_page_size=16384 --innodb_log_block_size=512 --innodb_undo_directory=. --innodb_undo_tablespaces=0 --server-id=0 --redo-log-version=0
xtrabackup: recognized client arguments: --prepare=1 --target-dir=/data/mysqlbackup/
xtrabackup version 2.4.15 based on MySQL server 5.7.19 Linux (x86_64) (revision id: 544842a)
xtrabackup: cd to /data/mysqlbackup/
xtrabackup: This target seems to be not prepared yet.
InnoDB: Number of pools: 1
xtrabackup: xtrabackup_logfile detected: size=8388608, start_lsn=(1633057)
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = .
xtrabackup:   innodb_log_files_in_group = 1
xtrabackup:   innodb_log_file_size = 8388608
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = .
xtrabackup:   innodb_log_files_in_group = 1
xtrabackup:   innodb_log_file_size = 8388608
xtrabackup: Starting InnoDB instance for recovery.
xtrabackup: Using 104857600 bytes for buffer pool (set by --use-memory parameter)
InnoDB: PUNCH HOLE support available
InnoDB: Mutexes and rw_locks use GCC atomic builtins
InnoDB: Uses event mutexes
InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
InnoDB: Compressed tables use zlib 1.2.7
InnoDB: Number of pools: 1
InnoDB: Using CPU crc32 instructions
InnoDB: Initializing buffer pool, total size = 100M, instances = 1, chunk size = 100M
InnoDB: Completed initialization of buffer pool
InnoDB: page_cleaner coordinator priority: -20
InnoDB: Highest supported file format is Barracuda.
InnoDB: The log sequence number 1625987 in the system tablespace does not match the log sequence number 1633057 in the ib_logfiles!
InnoDB: Database was not shutdown normally!
InnoDB: Starting crash recovery.
InnoDB: xtrabackup: Last MySQL binlog file position 4322, file name mysql-bin.000003
InnoDB: Creating shared tablespace for temporary tables
InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
InnoDB: File './ibtmp1' size is now 12 MB.
InnoDB: 96 redo rollback segment(s) found. 1 redo rollback segment(s) are active.
InnoDB: 32 non-redo rollback segment(s) are active.
InnoDB: 5.7.19 started; log sequence number 1633057
InnoDB: xtrabackup: Last MySQL binlog file position 4322, file name mysql-bin.000003

xtrabackup: starting shutdown with innodb_fast_shutdown = 1
InnoDB: FTS optimize thread exiting.
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 1633076
InnoDB: Number of pools: 1
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = .
xtrabackup:   innodb_log_files_in_group = 2
xtrabackup:   innodb_log_file_size = 50331648
InnoDB: PUNCH HOLE support available
InnoDB: Mutexes and rw_locks use GCC atomic builtins
InnoDB: Uses event mutexes
InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
InnoDB: Compressed tables use zlib 1.2.7
InnoDB: Number of pools: 1
InnoDB: Using CPU crc32 instructions
InnoDB: Initializing buffer pool, total size = 100M, instances = 1, chunk size = 100M
InnoDB: Completed initialization of buffer pool
InnoDB: page_cleaner coordinator priority: -20
InnoDB: Setting log file ./ib_logfile101 size to 48 MB
InnoDB: Setting log file ./ib_logfile1 size to 48 MB
InnoDB: Renaming log file ./ib_logfile101 to ./ib_logfile0
InnoDB: New log files created, LSN=1633076
InnoDB: Highest supported file format is Barracuda.
InnoDB: Log scan progressed past the checkpoint lsn 1633292
InnoDB: Doing recovery: scanned up to log sequence number 1633301 (0%)
InnoDB: Database was not shutdown normally!
InnoDB: Starting crash recovery.
InnoDB: xtrabackup: Last MySQL binlog file position 4322, file name mysql-bin.000003
InnoDB: Removed temporary tablespace data file: "ibtmp1"
InnoDB: Creating shared tablespace for temporary tables
InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
InnoDB: File './ibtmp1' size is now 12 MB.
InnoDB: 96 redo rollback segment(s) found. 1 redo rollback segment(s) are active.
InnoDB: 32 non-redo rollback segment(s) are active.
InnoDB: 5.7.19 started; log sequence number 1633301
xtrabackup: starting shutdown with innodb_fast_shutdown = 1
InnoDB: FTS optimize thread exiting.
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 1633320
190822 17:47:29 completed OK!
```


* 恢复

```bash
[root@localhost tmp]# xtrabackup --defaults-file=/usr/local/mysql/my.cnf --copy-back --target-dir=/data/mysqlbackup/
xtrabackup: recognized server arguments: --server-id=66154 --datadir=/usr/local/mysql/data --log_bin=mysql-bin
xtrabackup: recognized client arguments: --copy-back=1 --target-dir=/data/mysqlbackup/
xtrabackup version 2.4.15 based on MySQL server 5.7.19 Linux (x86_64) (revision id: 544842a)
190822 17:55:33 [01] Copying ib_logfile0 to /usr/local/mysql/data/ib_logfile0
190822 17:55:34 [01]        ...done
190822 17:55:34 [01] Copying ib_logfile1 to /usr/local/mysql/data/ib_logfile1
190822 17:55:34 [01]        ...done
190822 17:55:35 [01] Copying ibdata1 to /usr/local/mysql/data/ibdata1
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/innodb_table_stats.ibd to /usr/local/mysql/data/mysql/innodb_table_stats.ibd
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/innodb_index_stats.ibd to /usr/local/mysql/data/mysql/innodb_index_stats.ibd
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/slave_relay_log_info.ibd to /usr/local/mysql/data/mysql/slave_relay_log_info.ibd
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/slave_master_info.ibd to /usr/local/mysql/data/mysql/slave_master_info.ibd
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/slave_worker_info.ibd to /usr/local/mysql/data/mysql/slave_worker_info.ibd
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/db.frm to /usr/local/mysql/data/mysql/db.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/db.MYI to /usr/local/mysql/data/mysql/db.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/db.MYD to /usr/local/mysql/data/mysql/db.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/user.frm to /usr/local/mysql/data/mysql/user.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/user.MYI to /usr/local/mysql/data/mysql/user.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/user.MYD to /usr/local/mysql/data/mysql/user.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/func.frm to /usr/local/mysql/data/mysql/func.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/func.MYI to /usr/local/mysql/data/mysql/func.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/func.MYD to /usr/local/mysql/data/mysql/func.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/plugin.frm to /usr/local/mysql/data/mysql/plugin.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/plugin.MYI to /usr/local/mysql/data/mysql/plugin.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/plugin.MYD to /usr/local/mysql/data/mysql/plugin.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/servers.frm to /usr/local/mysql/data/mysql/servers.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/servers.MYI to /usr/local/mysql/data/mysql/servers.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/servers.MYD to /usr/local/mysql/data/mysql/servers.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/tables_priv.frm to /usr/local/mysql/data/mysql/tables_priv.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/tables_priv.MYI to /usr/local/mysql/data/mysql/tables_priv.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/tables_priv.MYD to /usr/local/mysql/data/mysql/tables_priv.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/columns_priv.frm to /usr/local/mysql/data/mysql/columns_priv.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/columns_priv.MYI to /usr/local/mysql/data/mysql/columns_priv.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/columns_priv.MYD to /usr/local/mysql/data/mysql/columns_priv.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_topic.frm to /usr/local/mysql/data/mysql/help_topic.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_topic.MYI to /usr/local/mysql/data/mysql/help_topic.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_topic.MYD to /usr/local/mysql/data/mysql/help_topic.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_category.frm to /usr/local/mysql/data/mysql/help_category.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_category.MYI to /usr/local/mysql/data/mysql/help_category.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_category.MYD to /usr/local/mysql/data/mysql/help_category.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_relation.frm to /usr/local/mysql/data/mysql/help_relation.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_relation.MYI to /usr/local/mysql/data/mysql/help_relation.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_relation.MYD to /usr/local/mysql/data/mysql/help_relation.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_keyword.frm to /usr/local/mysql/data/mysql/help_keyword.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_keyword.MYI to /usr/local/mysql/data/mysql/help_keyword.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/help_keyword.MYD to /usr/local/mysql/data/mysql/help_keyword.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_name.frm to /usr/local/mysql/data/mysql/time_zone_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_name.MYI to /usr/local/mysql/data/mysql/time_zone_name.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_name.MYD to /usr/local/mysql/data/mysql/time_zone_name.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone.frm to /usr/local/mysql/data/mysql/time_zone.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone.MYI to /usr/local/mysql/data/mysql/time_zone.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone.MYD to /usr/local/mysql/data/mysql/time_zone.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_transition.frm to /usr/local/mysql/data/mysql/time_zone_transition.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_transition.MYI to /usr/local/mysql/data/mysql/time_zone_transition.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_transition.MYD to /usr/local/mysql/data/mysql/time_zone_transition.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_transition_type.frm to /usr/local/mysql/data/mysql/time_zone_transition_type.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_transition_type.MYI to /usr/local/mysql/data/mysql/time_zone_transition_type.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_transition_type.MYD to /usr/local/mysql/data/mysql/time_zone_transition_type.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_leap_second.frm to /usr/local/mysql/data/mysql/time_zone_leap_second.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_leap_second.MYI to /usr/local/mysql/data/mysql/time_zone_leap_second.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/time_zone_leap_second.MYD to /usr/local/mysql/data/mysql/time_zone_leap_second.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/proc.frm to /usr/local/mysql/data/mysql/proc.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/proc.MYI to /usr/local/mysql/data/mysql/proc.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/proc.MYD to /usr/local/mysql/data/mysql/proc.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/procs_priv.frm to /usr/local/mysql/data/mysql/procs_priv.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/procs_priv.MYI to /usr/local/mysql/data/mysql/procs_priv.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/procs_priv.MYD to /usr/local/mysql/data/mysql/procs_priv.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/general_log.frm to /usr/local/mysql/data/mysql/general_log.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/general_log.CSM to /usr/local/mysql/data/mysql/general_log.CSM
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/general_log.CSV to /usr/local/mysql/data/mysql/general_log.CSV
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/slow_log.frm to /usr/local/mysql/data/mysql/slow_log.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/slow_log.CSM to /usr/local/mysql/data/mysql/slow_log.CSM
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/slow_log.CSV to /usr/local/mysql/data/mysql/slow_log.CSV
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/event.frm to /usr/local/mysql/data/mysql/event.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/event.MYI to /usr/local/mysql/data/mysql/event.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/event.MYD to /usr/local/mysql/data/mysql/event.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/ndb_binlog_index.frm to /usr/local/mysql/data/mysql/ndb_binlog_index.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/ndb_binlog_index.MYI to /usr/local/mysql/data/mysql/ndb_binlog_index.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/ndb_binlog_index.MYD to /usr/local/mysql/data/mysql/ndb_binlog_index.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/innodb_table_stats.frm to /usr/local/mysql/data/mysql/innodb_table_stats.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/innodb_index_stats.frm to /usr/local/mysql/data/mysql/innodb_index_stats.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/slave_relay_log_info.frm to /usr/local/mysql/data/mysql/slave_relay_log_info.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/slave_master_info.frm to /usr/local/mysql/data/mysql/slave_master_info.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/slave_worker_info.frm to /usr/local/mysql/data/mysql/slave_worker_info.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/proxies_priv.frm to /usr/local/mysql/data/mysql/proxies_priv.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/proxies_priv.MYI to /usr/local/mysql/data/mysql/proxies_priv.MYI
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./mysql/proxies_priv.MYD to /usr/local/mysql/data/mysql/proxies_priv.MYD
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./ttt/t1.ibd to /usr/local/mysql/data/ttt/t1.ibd
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./ttt/db.opt to /usr/local/mysql/data/ttt/db.opt
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./ttt/t1.frm to /usr/local/mysql/data/ttt/t1.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/db.opt to /usr/local/mysql/data/performance_schema/db.opt
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/cond_instances.frm to /usr/local/mysql/data/performance_schema/cond_instances.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_waits_current.frm to /usr/local/mysql/data/performance_schema/events_waits_current.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_waits_history.frm to /usr/local/mysql/data/performance_schema/events_waits_history.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_waits_history_long.frm to /usr/local/mysql/data/performance_schema/events_waits_history_long.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_waits_summary_by_instance.frm to /usr/local/mysql/data/performance_schema/events_waits_summary_by_instance.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_waits_summary_by_host_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_waits_summary_by_host_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_waits_summary_by_user_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_waits_summary_by_user_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_waits_summary_by_account_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_waits_summary_by_account_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_waits_summary_by_thread_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_waits_summary_by_thread_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_waits_summary_global_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_waits_summary_global_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/file_instances.frm to /usr/local/mysql/data/performance_schema/file_instances.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/file_summary_by_event_name.frm to /usr/local/mysql/data/performance_schema/file_summary_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/file_summary_by_instance.frm to /usr/local/mysql/data/performance_schema/file_summary_by_instance.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/socket_instances.frm to /usr/local/mysql/data/performance_schema/socket_instances.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/socket_summary_by_instance.frm to /usr/local/mysql/data/performance_schema/socket_summary_by_instance.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/socket_summary_by_event_name.frm to /usr/local/mysql/data/performance_schema/socket_summary_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/host_cache.frm to /usr/local/mysql/data/performance_schema/host_cache.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/mutex_instances.frm to /usr/local/mysql/data/performance_schema/mutex_instances.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/objects_summary_global_by_type.frm to /usr/local/mysql/data/performance_schema/objects_summary_global_by_type.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/performance_timers.frm to /usr/local/mysql/data/performance_schema/performance_timers.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/rwlock_instances.frm to /usr/local/mysql/data/performance_schema/rwlock_instances.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/setup_actors.frm to /usr/local/mysql/data/performance_schema/setup_actors.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/setup_consumers.frm to /usr/local/mysql/data/performance_schema/setup_consumers.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/setup_instruments.frm to /usr/local/mysql/data/performance_schema/setup_instruments.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/setup_objects.frm to /usr/local/mysql/data/performance_schema/setup_objects.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/setup_timers.frm to /usr/local/mysql/data/performance_schema/setup_timers.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/table_io_waits_summary_by_index_usage.frm to /usr/local/mysql/data/performance_schema/table_io_waits_summary_by_index_usage.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/table_io_waits_summary_by_table.frm to /usr/local/mysql/data/performance_schema/table_io_waits_summary_by_table.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/table_lock_waits_summary_by_table.frm to /usr/local/mysql/data/performance_schema/table_lock_waits_summary_by_table.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/threads.frm to /usr/local/mysql/data/performance_schema/threads.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_stages_current.frm to /usr/local/mysql/data/performance_schema/events_stages_current.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_stages_history.frm to /usr/local/mysql/data/performance_schema/events_stages_history.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_stages_history_long.frm to /usr/local/mysql/data/performance_schema/events_stages_history_long.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_stages_summary_by_thread_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_stages_summary_by_thread_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_stages_summary_by_host_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_stages_summary_by_host_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_stages_summary_by_user_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_stages_summary_by_user_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_stages_summary_by_account_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_stages_summary_by_account_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_stages_summary_global_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_stages_summary_global_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_statements_current.frm to /usr/local/mysql/data/performance_schema/events_statements_current.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_statements_history.frm to /usr/local/mysql/data/performance_schema/events_statements_history.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_statements_history_long.frm to /usr/local/mysql/data/performance_schema/events_statements_history_long.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_statements_summary_by_thread_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_statements_summary_by_thread_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_statements_summary_by_host_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_statements_summary_by_host_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_statements_summary_by_user_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_statements_summary_by_user_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_statements_summary_by_account_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_statements_summary_by_account_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_statements_summary_global_by_event_name.frm to /usr/local/mysql/data/performance_schema/events_statements_summary_global_by_event_name.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/hosts.frm to /usr/local/mysql/data/performance_schema/hosts.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/users.frm to /usr/local/mysql/data/performance_schema/users.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/accounts.frm to /usr/local/mysql/data/performance_schema/accounts.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/events_statements_summary_by_digest.frm to /usr/local/mysql/data/performance_schema/events_statements_summary_by_digest.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/session_connect_attrs.frm to /usr/local/mysql/data/performance_schema/session_connect_attrs.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./performance_schema/session_account_connect_attrs.frm to /usr/local/mysql/data/performance_schema/session_account_connect_attrs.frm
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./xtrabackup_info to /usr/local/mysql/data/xtrabackup_info
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./xtrabackup_binlog_pos_innodb to /usr/local/mysql/data/xtrabackup_binlog_pos_innodb
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./xtrabackup_master_key_id to /usr/local/mysql/data/xtrabackup_master_key_id
190822 17:55:35 [01]        ...done
190822 17:55:35 [01] Copying ./ibtmp1 to /usr/local/mysql/data/ibtmp1
190822 17:55:35 [01]        ...done
190822 17:55:35 completed OK!
```

* 启动服务

```bash
[root@localhost mysql]# chown -R mysql:mysql /usr/local/mysql/data
[root@localhost mysql]# systemctl start mysqld
```

