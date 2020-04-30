
---
title: 实践-中间件ProxySQL使用
date: 2020/04/30
tags: 
  mysql
---
  

## 环境
|OS          |Kernel   |    Software    |
|:-----------|:-----   | :-----------   | 
|CentOS 7.3  |3.10.0   | 5.7.21-21-log Percona Server |
 

## 先决条件
* 确保你有一组运行中的MGR组复制集群或GTID复制集群

## 注意事项
* 无

## ProxySQL安装

* 下载安装包

```bash
[root@qs-storage-swift-01 src]#  wget -S https://github.com/sysown/proxysql/releases/download/v2.0.10/proxysql-2.0.10-1-centos7.x86_64.rpm 
```

* 安装RPM包

```bash
[root@qs-storage-swift-01 src]# rpm -ivh proxysql-2.0.10-1-centos7.x86_64.rpm 
Preparing...                          ################################# [100%]
Updating / installing...
   1:proxysql-2.0.10-1                ################################# [100%]
[root@qs-storage-swift-01 src]# 
[root@qs-storage-swift-01 src]# rpm -ql proxysql
/etc/logrotate.d/proxysql
/etc/proxysql.cnf
/etc/systemd/system/proxysql.service
/usr/bin/proxysql
/usr/share/proxysql/tools/proxysql_galera_checker.sh
/usr/share/proxysql/tools/proxysql_galera_writer.pl
```

* 启动proxysql服务

```bash
[root@qs-storage-swift-01 ~]# systemctl  start proxysql 
[root@qs-storage-swift-01 ~]# systemctl  status  proxysql 
● proxysql.service - High Performance Advanced Proxy for MySQL
   Loaded: loaded (/etc/systemd/system/proxysql.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-04-29 10:13:06 CST; 15s ago
  Process: 24465 ExecStart=/usr/bin/proxysql -c /etc/proxysql.cnf (code=exited, status=0/SUCCESS)
 Main PID: 24467 (proxysql)
   Memory: 87.3M
   CGroup: /system.slice/proxysql.service
           ├─24467 /usr/bin/proxysql -c /etc/proxysql.cnf
           └─24468 /usr/bin/proxysql -c /etc/proxysql.cnf

Apr 29 10:13:06 qs-storage-swift-01.localdomain systemd[1]: Starting High Performance Advanced Proxy for MySQL...
Apr 29 10:13:06 qs-storage-swift-01.localdomain proxysql[24465]: 2020-04-29 10:13:06 [INFO] Using config file /etc/proxysql.cnf
Apr 29 10:13:06 qs-storage-swift-01.localdomain proxysql[24465]: 2020-04-29 10:13:06 [INFO] Using OpenSSL version: OpenSSL 1.1.0h  27 Mar 2018
Apr 29 10:13:06 qs-storage-swift-01.localdomain proxysql[24465]: 2020-04-29 10:13:06 [INFO] No SSL keys/certificates found in datadir (/var/lib/proxysql). Generating new keys/certificates.
Apr 29 10:13:06 qs-storage-swift-01.localdomain systemd[1]: Started High Performance Advanced Proxy for MySQL.
You have mail in /var/spool/mail/root
[root@qs-storage-swift-01 ~]# 
```

* 登陆到proxysql

```bash
[root@qs-storage-swift-01 ~]# mysql -u admin -padmin -h 127.0.0.1 -P 6032 --prompt='Admin>' 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.5.30 (ProxySQL Admin Module)

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

Admin>show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | disk          | /var/lib/proxysql/proxysql.db       |
| 3   | stats         |                                     |
| 4   | monitor       |                                     |
| 5   | stats_history | /var/lib/proxysql/proxysql_stats.db |
+-----+---------------+-------------------------------------+
5 rows in set (0.00 sec)

Admin>show tables;
+----------------------------------------------------+
| tables                                             |
+----------------------------------------------------+
| global_variables                                   |
| mysql_aws_aurora_hostgroups                        |
| mysql_collations                                   |
| mysql_firewall_whitelist_rules                     |
| mysql_firewall_whitelist_sqli_fingerprints         |
| mysql_firewall_whitelist_users                     |
| mysql_galera_hostgroups                            |
| mysql_group_replication_hostgroups                 |
| mysql_query_rules                                  |
| mysql_query_rules_fast_routing                     |
| mysql_replication_hostgroups                       |
| mysql_servers                                      |
| mysql_users                                        |
| proxysql_servers                                   |
| restapi_routes                                     |
| runtime_checksums_values                           |
| runtime_global_variables                           |
| runtime_mysql_aws_aurora_hostgroups                |
| runtime_mysql_firewall_whitelist_rules             |
| runtime_mysql_firewall_whitelist_sqli_fingerprints |
| runtime_mysql_firewall_whitelist_users             |
| runtime_mysql_galera_hostgroups                    |
| runtime_mysql_group_replication_hostgroups         |
| runtime_mysql_query_rules                          |
| runtime_mysql_query_rules_fast_routing             |
| runtime_mysql_replication_hostgroups               |
| runtime_mysql_servers                              |
| runtime_mysql_users                                |
| runtime_proxysql_servers                           |
| runtime_restapi_routes                             |
| runtime_scheduler                                  |
| scheduler                                          |
+----------------------------------------------------+
32 rows in set (0.00 sec)

Admin>

```

## ProxySQL实践

#### 代理普通GTID复制,实现读写分离、健康检测

* 环境

|OS          |IP       |   Role   |  Version |
|:-----------|:-----   | :-----   | :--------|
|CentOS 7.3  |10.99.66.152:3307  |  Master| 5.7.21-21-log Percona Server|
|CentOS 7.3  |10.99.66.153:3307  |  Slave | 5.7.21-21-log Percona Server|
|CentOS 7.3  |10.99.66.153:3308  |  Slave | 5.7.21-21-log Percona Server|

* 在后端MYSQL创建业务使用的账户，这里以app为例子,在Master执行即可。

```bash
MySQL [mysql]> grant all on *.* to app@'%' identified by 'app';
Query OK, 0 rows affected, 1 warning (0.00 sec)

MySQL [mysql]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MySQL [mysql]> show grants for app;
+------------------------------------------+
| Grants for app@%                         |
+------------------------------------------+
| GRANT ALL PRIVILEGES ON *.* TO 'app'@'%' |
+------------------------------------------+
1 row in set (0.00 sec)

MySQL [mysql]>
```

* 在后端MYSQL的所有从库配置，开启read_only选项。

```bash
[root@localhost][(none)]>  set global read_only = 1;
```

* 配置proxysql,添加所有MYSQL服务器

NOTE：ProxySQL使用组来管理MYSQL服务器具，分别为写组、备写组、读组、离线组，这里使用（10写 20备写 30读 40离线）来区分。

```bash
[root@prometheus-shqs-1 proxysql]# mysql -u admin -padmin -h 127.0.0.1 -P 6032 --prompt='Admin>' 
Admin>insert into mysql_servers (hostgroup_id,hostname,port,max_connections,comment) values (10,'10.99.66.152',3307,2000,'GTID-Replication');
Query OK, 1 row affected (0.00 sec)

Admin>insert into mysql_servers (hostgroup_id,hostname,port,max_connections,comment) values (10,'10.99.66.153',3307,2000,'GTID-Replication');
Query OK, 1 row affected (0.00 sec)

Admin>insert into mysql_servers (hostgroup_id,hostname,port,max_connections,comment) values (10,'10.99.66.153',3308,2000,'GTID-Replication');
Query OK, 1 row affected (0.00 sec)

Admin>load mysql servers to runtime;   //保存配置到运行中
Query OK, 0 rows affected (0.01 sec)

Admin>save  mysql servers to disk ;    //保存配置到磁盘
Query OK, 0 rows affected (0.03 sec)

Admin>select * from runtime_mysql_servers;
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+------------------+
| hostgroup_id | hostname     | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment          |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+------------------+
| 10           | 10.99.66.152 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | GTID-Replication |
| 10           | 10.99.66.153 | 3308 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | GTID-Replication |
| 10           | 10.99.66.153 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | GTID-Replication |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+------------------+
3 rows in set (0.00 sec)

Admin>
```

* 配置proxysql,指定健康检测运行账户（监控后端MYSQL是否存活的）

```bash
Admin> set mysql-monitor_username = 'app';   //这里为了简单复用业务账户app
Query OK, 1 row affected (0.00 sec)

Admin>set mysql-monitor_password = 'app';
Query OK, 1 row affected (0.00 sec)

Admin>load mysql variables to runtime;       //保存配置到运行中
Query OK, 0 rows affected (0.00 sec)

Admin>save mysql variables to disk;          //保存配置到磁盘
Query OK, 143 rows affected (0.00 sec)

Admin>
Admin>insert into mysql_users (username,password,active,default_hostgroup) values ('app','app',1,30);     //插入app用户到proxysql用户表里
Query OK, 1 row affected (0.01 sec)

Admin>load mysql users to runtime;           //保存配置到运行中
Query OK, 0 rows affected (0.00 sec)

Admin>save mysql users to disk;
Query OK, 0 rows affected (0.00 sec)         //保存配置到磁盘
```

* 配置proxysql,添加GTID复制配置

```bash
Admin>insert into mysql_replication_hostgroups  values (10,30,'read_only','GTID-Replication'); //这里10组为写组，30为读组
Query OK, 1 row affected (0.00 sec)

Admin>load mysql variables to runtime;       //保存配置到运行中
Query OK, 0 rows affected (0.00 sec)

Admin>save mysql variables to disk;          //保存配置到磁盘
Query OK, 143 rows affected (0.01 sec)

Admin>
Admin>select * from  mysql_replication_hostgroups ;
+------------------+------------------+------------+------------------+
| writer_hostgroup | reader_hostgroup | check_type | comment          |
+------------------+------------------+------------+------------------+
| 10               | 30               | read_only  | GTID-Replication |
+------------------+------------------+------------+------------------+
1 row in set (0.01 sec)
```

* 配置proxysql,配置读写分离策略

```bash
Admin>
Admin>insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply)  values (1,1,'^SELECT.*FOR UPDATE$',10,1);
Admin>insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply)  values (2,1,'^SELECT',30,1); 
Query OK, 1 row affected (0.00 sec)

Admin>load mysql query rules to runtime;
Query OK, 0 rows affected (0.01 sec)

Admin>save mysql query rules to disk;
Query OK, 0 rows affected (0.01 sec)

Admin> select * from runtime_mysql_query_rules \G
*************************** 1. row ***************************
              rule_id: 1
               active: 1
             username: NULL
           schemaname: NULL
               flagIN: 0
          client_addr: NULL
           proxy_addr: NULL
           proxy_port: NULL
               digest: NULL
         match_digest: ^SELECT.*FOR UPDATE$
        match_pattern: NULL
 negate_match_pattern: 0
         re_modifiers: CASELESS
              flagOUT: NULL
      replace_pattern: NULL
destination_hostgroup: 10
            cache_ttl: NULL
   cache_empty_result: NULL
        cache_timeout: NULL
            reconnect: NULL
              timeout: NULL
              retries: NULL
                delay: NULL
    next_query_flagIN: NULL
       mirror_flagOUT: NULL
     mirror_hostgroup: NULL
            error_msg: NULL
               OK_msg: NULL
          sticky_conn: NULL
            multiplex: NULL
  gtid_from_hostgroup: NULL
                  log: NULL
                apply: 1
              comment: NULL
*************************** 2. row ***************************
              rule_id: 2
               active: 1
             username: NULL
           schemaname: NULL
               flagIN: 0
          client_addr: NULL
           proxy_addr: NULL
           proxy_port: NULL
               digest: NULL
         match_digest: ^SELECT
        match_pattern: NULL
 negate_match_pattern: 0
         re_modifiers: CASELESS
              flagOUT: NULL
      replace_pattern: NULL
destination_hostgroup: 30
            cache_ttl: NULL
   cache_empty_result: NULL
        cache_timeout: NULL
            reconnect: NULL
              timeout: NULL
              retries: NULL
                delay: NULL
    next_query_flagIN: NULL
       mirror_flagOUT: NULL
     mirror_hostgroup: NULL
            error_msg: NULL
               OK_msg: NULL
          sticky_conn: NULL
            multiplex: NULL
  gtid_from_hostgroup: NULL
                  log: NULL
                apply: 1
              comment: NULL
2 rows in set (0.01 sec)
```

* 查看proxysql对GTID复制的默认组分配情况

```bash
Admin>select * from runtime_mysql_servers;   //这是根据以上的配置之后，proxysql自动为我们分配的
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+------------------+
| hostgroup_id | hostname     | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment          |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+------------------+
| 10           | 10.99.66.152 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | GTID-Replication |
| 30           | 10.99.66.153 | 3308 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | GTID-Replication |
| 30           | 10.99.66.153 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | GTID-Replication |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+------------------+
3 rows in set (0.01 sec)

Admin>
```

* 验证读写分离

```bash
Admin>select * from stats_mysql_query_digest  \G
*************************** 1. row ***************************   //查询数据默认分配到30这个读组
        hostgroup: 30
       schemaname: information_schema
         username: app
   client_address: 
           digest: 0xF26CE0C705616F8D
      digest_text: select * from zltest01.t1
       count_star: 1
       first_seen: 1588233465
        last_seen: 1588233465
         sum_time: 2211
         min_time: 2211
         max_time: 2211
sum_rows_affected: 0
    sum_rows_sent: 2
*************************** 8. row ***************************     //创建库的确分配到10这个写组
        hostgroup: 10
       schemaname: information_schema
         username: app
   client_address: 
           digest: 0x1B0A8DAC71110784
      digest_text: create database tade002
       count_star: 1
       first_seen: 1588218431
        last_seen: 1588218431
         sum_time: 3500
         min_time: 3500
         max_time: 3500
sum_rows_affected: 1
    sum_rows_sent: 0
```

#### 代理组复制-单主模式（实现HA高可用、健康检测）

* 环境

|OS          |IP       |   Role   |  Version |
|:-----------|:-----   | :-----   | :--------|
|CentOS 7.3  |10.99.72.4:3307   |  Master | 5.7.21-21-log Percona Server|
|CentOS 7.3  |10.99.72.5:3307   |  Slave  | 5.7.21-21-log Percona Server|
|CentOS 7.3  |10.99.72.6:3307   |  Slave  | 5.7.21-21-log Percona Server|

* 由于ProxySQL需要访问一个MGR视图，所以需要在后端MYSLQ集群加载视图脚本

```bash
[root@qs-storage-swift-01 3307_test]# cat addition_to_sys.sql
USE sys;

DELIMITER $$

CREATE FUNCTION IFZERO(a INT, b INT)
RETURNS INT
DETERMINISTIC
RETURN IF(a = 0, b, a)$$

CREATE FUNCTION LOCATE2(needle TEXT(10000), haystack TEXT(10000), offset INT)
RETURNS INT
DETERMINISTIC
RETURN IFZERO(LOCATE(needle, haystack, offset), LENGTH(haystack) + 1)$$

CREATE FUNCTION GTID_NORMALIZE(g TEXT(10000))
RETURNS TEXT(10000)
DETERMINISTIC
RETURN GTID_SUBTRACT(g, '')$$

CREATE FUNCTION GTID_COUNT(gtid_set TEXT(10000))
RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE result BIGINT DEFAULT 0;
  DECLARE colon_pos INT;
  DECLARE next_dash_pos INT;
  DECLARE next_colon_pos INT;
  DECLARE next_comma_pos INT;
  SET gtid_set = GTID_NORMALIZE(gtid_set);
  SET colon_pos = LOCATE2(':', gtid_set, 1);
  WHILE colon_pos != LENGTH(gtid_set) + 1 DO
     SET next_dash_pos = LOCATE2('-', gtid_set, colon_pos + 1);
     SET next_colon_pos = LOCATE2(':', gtid_set, colon_pos + 1);
     SET next_comma_pos = LOCATE2(',', gtid_set, colon_pos + 1);
     IF next_dash_pos < next_colon_pos AND next_dash_pos < next_comma_pos THEN
       SET result = result +
         SUBSTR(gtid_set, next_dash_pos + 1,
                LEAST(next_colon_pos, next_comma_pos) - (next_dash_pos + 1)) -
         SUBSTR(gtid_set, colon_pos + 1, next_dash_pos - (colon_pos + 1)) + 1;
     ELSE
       SET result = result + 1;
     END IF;
     SET colon_pos = next_colon_pos;
  END WHILE;
  RETURN result;
END$$

CREATE FUNCTION gr_applier_queue_length()
RETURNS INT
DETERMINISTIC
BEGIN
  RETURN (SELECT sys.gtid_count( GTID_SUBTRACT( (SELECT
Received_transaction_set FROM performance_schema.replication_connection_status
WHERE Channel_name = 'group_replication_applier' ), (SELECT
@@global.GTID_EXECUTED) )));
END$$

CREATE FUNCTION gr_member_in_primary_partition()
RETURNS VARCHAR(3)
DETERMINISTIC
BEGIN
  RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
performance_schema.replication_group_members WHERE MEMBER_STATE != 'ONLINE') >=
((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
performance_schema.replication_group_member_stats USING(member_id));
END$$

CREATE VIEW gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
sys.gr_applier_queue_length() as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert' from performance_schema.replication_group_member_stats;$$

DELIMITER ;
[root@qs-storage-swift-01 3307_test]# mysql -S mysql.sock
[root@localhost][sys]>  source addition_to_sys.sql ;                                 //在集群内Master节点执行即可
[root@localhost][sys]> 
[root@localhost][sys]> select * from  sys.gr_member_routing_candidate_status ;       //Master上的状态
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | NO        |                   0 |                    0 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.01 sec)

[root@localhost][sys]>
[root@localhost][sys]> select * from  gr_member_routing_candidate_status ;            //Slave上的状态,MGR从节点read_only会自动设上
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | YES       |                   0 |                    0 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.01 sec)

[root@localhost][sys]>
```

* 配置proxysql,添加所有MYSQL服务器

NOTE：ProxySQL使用组来管理MYSQL服务器集群，分别为写组、备写组、读组、离线组，这里使用（10写 20备写 30读 40离线）来区分。

```bash
[root@prometheus-shqs-1 ~]# mysql -u admin -padmin -h 127.0.0.1 -P 6032 --prompt='Admin>'
Admin>
Admin>insert into mysql_servers (hostgroup_id,hostname,port,max_connections,comment) values (10,'10.99.72.4',3307,2000,'MGR-Cluster'); 
Query OK, 1 row affected (0.00 sec)

Admin>insert into mysql_servers (hostgroup_id,hostname,port,max_connections,comment) values (10,'10.99.72.5',3307,2000,'MGR-Cluster'); 
Query OK, 1 row affected (0.00 sec)

Admin>insert into mysql_servers (hostgroup_id,hostname,port,max_connections,comment) values (10,'10.99.72.6',3307,2000,'MGR-Cluster'); 
Query OK, 1 row affected (0.00 sec)

Admin>load mysql servers to runtime;          //保存配置到运行中
Query OK, 0 rows affected (0.01 sec)

Admin>save mysql servers to disk;             //保存配置到磁盘
Query OK, 0 rows affected (0.01 sec) 

Admin>select * from runtime_mysql_servers;
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| hostgroup_id | hostname   | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment     |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| 10           | 10.99.72.4 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 10           | 10.99.72.6 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 10           | 10.99.72.5 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
3 rows
```

* 配置proxysql,添加MGR组复制的信息

```bash

Admin>insert into mysql_group_replication_hostgroups (writer_hostgroup,backup_writer_hostgroup,reader_hostgroup,offline_hostgroup,active,max_writers,writer_is_also_reader,max_transactions_behind) values (10,20,30,40,1,1,0,0);  
Query OK, 1 row affected (0.00 sec)

Admin>load mysql servers to runtime;
Query OK, 0 rows affected (0.00 sec)

Admin>save mysql servers to disk;
Query OK, 0 rows affected (0.02 sec)

Admin>select * from mysql_group_replication_hostgroups;
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| writer_hostgroup | backup_writer_hostgroup | reader_hostgroup | offline_hostgroup | active | max_writers | writer_is_also_reader | max_transactions_behind | comment |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| 10               | 20                      | 30               | 40                | 1      | 1           | 0                     | 0                       | NULL    |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
1 row in set (0.00 sec)

Admin>
```

* 在后端MYSQL创建业务使用的账户以及ProxySQL监控使用的账户，这里以app和monitor为例子,任意节点执行

```bash
[root@localhost][mysql]> grant all on *.* to app@'%' identified by 'app';                    //实际应用程序使用的账户
Query OK, 0 rows affected, 1 warning (0.00 sec)

[root@localhost][mysql]> GRANT SELECT ON `sys`.* TO 'monitor'@'%'  identified by 'monitor';   //proxysql监控MGR集群使用的账户
Query OK, 0 rows affected, 1 warning (0.00 sec)

[root@localhost][mysql]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

[root@localhost][mysql]>
```


* 配置proxysql,配置监控账户、业务读写账户

```bash
Admin>
Admin>set mysql-monitor_username = 'monitor';               //配置监控账户
Query OK, 1 row affected (0.00 sec)

Admin>set mysql-monitor_password = 'monitor';
Query OK, 1 row affected (0.00 sec)

Admin>load mysql variables to runtime;
Query OK, 0 rows affected (0.01 sec)

Admin>save mysql variables to disk;
Query OK, 143 rows affected (0.00 sec)

Admin>insert into mysql_users (username,password,active,default_hostgroup) values ('app','app',1,10);    //配置实际业务读写账户
Query OK, 1 row affected (0.00 sec)

Admin>load mysql users to runtime;
Query OK, 0 rows affected (0.00 sec)

Admin>save mysql users to disk; 
Query OK, 0 rows affected (0.01 se

Admin>select username,password,active,default_hostgroup from runtime_mysql_users;
+----------+-------------------------------------------+--------+-------------------+
| username | password                                  | active | default_hostgroup |
+----------+-------------------------------------------+--------+-------------------+
| app      | *5BCB3E6AC345B435C7C2E6B7949A04CE6F6563D3 | 1      | 10                |
| app      | *5BCB3E6AC345B435C7C2E6B7949A04CE6F6563D3 | 1      | 10                |
+----------+-------------------------------------------+--------+-------------------+
2 rows in set (0.00 sec)

Admin>
```

* 验证proxysql的后端mysql分配组规则 

```bash
Admin>
Admin>select * from runtime_mysql_servers;  //可以看到72.5和72.6自动被分配到了30组（读组）里了。
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| hostgroup_id | hostname   | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment     |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| 10           | 10.99.72.4 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 30           | 10.99.72.5 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 30           | 10.99.72.6 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
3 rows in set (0.00 sec)

Admin>
```

* 配置proxysql读写分离规则

```bash
Admin>insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply) values (1,1,'^SELECT.*FOR UPDATE$',10,1),(2,1,'^SELECT',30,1);
Query OK, 2 rows affected (0.01 sec)

Admin>load mysql query rules to runtime;
Query OK, 0 rows affected (0.00 sec)

Admin>save mysql query rules to disk;
Query OK, 0 rows affected (0.02 sec)

Admin>select * from runtime_mysql_query_rules\G
*************************** 1. row ***************************
              rule_id: 1
               active: 1
             username: NULL
           schemaname: NULL
               flagIN: 0
          client_addr: NULL
           proxy_addr: NULL
           proxy_port: NULL
               digest: NULL
         match_digest: ^SELECT.*FOR UPDATE$
        match_pattern: NULL
 negate_match_pattern: 0
         re_modifiers: CASELESS
              flagOUT: NULL
      replace_pattern: NULL
destination_hostgroup: 10
            cache_ttl: NULL
   cache_empty_result: NULL
        cache_timeout: NULL
            reconnect: NULL
              timeout: NULL
              retries: NULL
                delay: NULL
    next_query_flagIN: NULL
       mirror_flagOUT: NULL
     mirror_hostgroup: NULL
            error_msg: NULL
               OK_msg: NULL
          sticky_conn: NULL
            multiplex: NULL
  gtid_from_hostgroup: NULL
                  log: NULL
                apply: 1
              comment: NULL
*************************** 2. row ***************************
              rule_id: 2
               active: 1
             username: NULL
           schemaname: NULL
               flagIN: 0
          client_addr: NULL
           proxy_addr: NULL
           proxy_port: NULL
               digest: NULL
         match_digest: ^SELECT
        match_pattern: NULL
 negate_match_pattern: 0
         re_modifiers: CASELESS
              flagOUT: NULL
      replace_pattern: NULL
destination_hostgroup: 30
            cache_ttl: NULL
   cache_empty_result: NULL
        cache_timeout: NULL
            reconnect: NULL
              timeout: NULL
              retries: NULL
                delay: NULL
    next_query_flagIN: NULL
       mirror_flagOUT: NULL
     mirror_hostgroup: NULL
            error_msg: NULL
               OK_msg: NULL
          sticky_conn: NULL
            multiplex: NULL
  gtid_from_hostgroup: NULL
                  log: NULL
                apply: 1
              comment: NULL
2 rows in set (0.00 sec)

Admin>
```

* 验证读写分离

```bash
Admin> select * from stats_mysql_query_digest\G      //可以看到还是很均衡的，select分配到了30组，即为读组；create就分配到默认组，10写组。
*************************** 1. row ***************************
        hostgroup: 10
       schemaname: information_schema
         username: app
   client_address: 
           digest: 0xAFDBD733C1755006
      digest_text: create database dbtest02
       count_star: 1
       first_seen: 1588242932
        last_seen: 1588242932
         sum_time: 2267
         min_time: 2267
         max_time: 2267
sum_rows_affected: 1
    sum_rows_sent: 0
*************************** 2. row ***************************
        hostgroup: 30
       schemaname: information_schema
         username: app
   client_address: 
           digest: 0x681435217136D007
      digest_text: select * from dbtest01.t1
       count_star: 1
       first_seen: 1588242899
        last_seen: 1588242899
         sum_time: 3879
         min_time: 3879
         max_time: 3879
sum_rows_affected: 0
    sum_rows_sent: 1
```

* 验证proxysql高可用能力

```bash
Admin> select * from runtime_mysql_servers;     //当前Master是10.99.72.4，我们马上关闭
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| hostgroup_id | hostname   | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment     |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| 10           | 10.99.72.4 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 30           | 10.99.72.5 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 30           | 10.99.72.6 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
3 rows in set (0.00 sec)

-----10.99.72.4 已关闭

Admin> select * from runtime_mysql_servers;     //可以看到10.99.72.4已经被放置到40离线组中去了，10组里72.6已经顶替上去。 
+--------------+------------+------+-----------+---------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| hostgroup_id | hostname   | port | gtid_port | status  | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment     |
+--------------+------------+------+-----------+---------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| 10           | 10.99.72.6 | 3307 | 0         | ONLINE  | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 40           | 10.99.72.4 | 3307 | 0         | SHUNNED | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 30           | 10.99.72.5 | 3307 | 0         | ONLINE  | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
+--------------+------------+------+-----------+---------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
3 rows in set (0.00 sec)

Admin>
Admin>
```

#### 代理租复制-多主模式（实现HA高可用、健康检测）


* 环境

|OS          |IP       |   Role   |  Version |
|:-----------|:-----   | :-----   | :--------|
|CentOS 7.3  |10.99.72.4:3307   |  Master | 5.7.21-21-log Percona Server|
|CentOS 7.3  |10.99.72.5:3307   |  Master | 5.7.21-21-log Percona Server|
|CentOS 7.3  |10.99.72.6:3307   |  Master | 5.7.21-21-log Percona Server|


* 由于ProxySQL需要访问一个MGR视图，所以需要在后端MYSLQ集群加载视图脚本

```bash
[root@qs-storage-swift-01 3307_test]# cat addition_to_sys.sql
USE sys;

DELIMITER $$

CREATE FUNCTION IFZERO(a INT, b INT)
RETURNS INT
DETERMINISTIC
RETURN IF(a = 0, b, a)$$

CREATE FUNCTION LOCATE2(needle TEXT(10000), haystack TEXT(10000), offset INT)
RETURNS INT
DETERMINISTIC
RETURN IFZERO(LOCATE(needle, haystack, offset), LENGTH(haystack) + 1)$$

CREATE FUNCTION GTID_NORMALIZE(g TEXT(10000))
RETURNS TEXT(10000)
DETERMINISTIC
RETURN GTID_SUBTRACT(g, '')$$

CREATE FUNCTION GTID_COUNT(gtid_set TEXT(10000))
RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE result BIGINT DEFAULT 0;
  DECLARE colon_pos INT;
  DECLARE next_dash_pos INT;
  DECLARE next_colon_pos INT;
  DECLARE next_comma_pos INT;
  SET gtid_set = GTID_NORMALIZE(gtid_set);
  SET colon_pos = LOCATE2(':', gtid_set, 1);
  WHILE colon_pos != LENGTH(gtid_set) + 1 DO
     SET next_dash_pos = LOCATE2('-', gtid_set, colon_pos + 1);
     SET next_colon_pos = LOCATE2(':', gtid_set, colon_pos + 1);
     SET next_comma_pos = LOCATE2(',', gtid_set, colon_pos + 1);
     IF next_dash_pos < next_colon_pos AND next_dash_pos < next_comma_pos THEN
       SET result = result +
         SUBSTR(gtid_set, next_dash_pos + 1,
                LEAST(next_colon_pos, next_comma_pos) - (next_dash_pos + 1)) -
         SUBSTR(gtid_set, colon_pos + 1, next_dash_pos - (colon_pos + 1)) + 1;
     ELSE
       SET result = result + 1;
     END IF;
     SET colon_pos = next_colon_pos;
  END WHILE;
  RETURN result;
END$$

CREATE FUNCTION gr_applier_queue_length()
RETURNS INT
DETERMINISTIC
BEGIN
  RETURN (SELECT sys.gtid_count( GTID_SUBTRACT( (SELECT
Received_transaction_set FROM performance_schema.replication_connection_status
WHERE Channel_name = 'group_replication_applier' ), (SELECT
@@global.GTID_EXECUTED) )));
END$$

CREATE FUNCTION gr_member_in_primary_partition()
RETURNS VARCHAR(3)
DETERMINISTIC
BEGIN
  RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
performance_schema.replication_group_members WHERE MEMBER_STATE != 'ONLINE') >=
((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
performance_schema.replication_group_member_stats USING(member_id));
END$$

CREATE VIEW gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
sys.gr_applier_queue_length() as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert' from performance_schema.replication_group_member_stats;$$

DELIMITER ;
[root@qs-storage-swift-01 3307_test]# mysql -S mysql.sock
[root@localhost][sys]>  source addition_to_sys.sql ;                             //集群内任意节点执行即可
[root@localhost][sys]> 
[root@localhost][sys]>  select * from gr_member_routing_candidate_status;        //查看组复制实际状态
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | NO        |                   0 |                    0 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.00 sec)

[root@localhost][sys]>
```

* 配置proxysql,添加所有MYSQL服务器

NOTE：ProxySQL使用组来管理MYSQL服务器集群，分别为写组、备写组、读组、离线组，这里使用（10写 20备写 30读 40离线）来区分。

```bash
[root@prometheus-shqs-1 ~]# mysql -u admin -padmin -h 127.0.0.1 -P 6032 --prompt='Admin>'
Admin>
Admin>insert into mysql_servers (hostgroup_id,hostname,port,max_connections,comment) values (10,'10.99.72.4',3307,2000,'MGR-Cluster'); 
Query OK, 1 row affected (0.00 sec)

Admin>insert into mysql_servers (hostgroup_id,hostname,port,max_connections,comment) values (10,'10.99.72.5',3307,2000,'MGR-Cluster'); 
Query OK, 1 row affected (0.00 sec)

Admin>insert into mysql_servers (hostgroup_id,hostname,port,max_connections,comment) values (10,'10.99.72.6',3307,2000,'MGR-Cluster'); 
Query OK, 1 row affected (0.00 sec)

Admin>load mysql servers to runtime;          //保存配置到运行中
Query OK, 0 rows affected (0.01 sec)

Admin>save mysql servers to disk;             //保存配置到磁盘
Query OK, 0 rows affected (0.01 sec) 

Admin>select * from runtime_mysql_servers;
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| hostgroup_id | hostname   | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment     |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| 10           | 10.99.72.4 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 10           | 10.99.72.6 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 10           | 10.99.72.5 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
3 rows
```

* 配置proxysql,添加MGR组复制的信息

```bash
Admin>insert into mysql_group_replication_hostgroups (writer_hostgroup,backup_writer_hostgroup,reader_hostgroup,offline_hostgroup,active,max_writers,writer_is_also_reader,max_transactions_behind) values (10,20,30,40,1,1,0,0);
Query OK, 1 row affected (0.00 sec)

Admin>load mysql servers to runtime;
Query OK, 0 rows affected (0.01 sec)

Admin>save mysql servers to disk;
Query OK, 0 rows affected (0.03 sec)

Admin>select * from runtime_mysql_group_replication_hostgroups ;    
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| writer_hostgroup | backup_writer_hostgroup | reader_hostgroup | offline_hostgroup | active | max_writers | writer_is_also_reader | max_transactions_behind | comment |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| 10               | 20                      | 30               | 40                | 1      | 1           | 0                     | 0                       | NULL    |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
1 row in set (0.00 sec)

Admin>
Admin> select * from runtime_mysql_servers;   //可以看到proxysql已经自动给分好组了。
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| hostgroup_id | hostname   | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment     |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| 10           | 10.99.72.6 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 20           | 10.99.72.5 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 20           | 10.99.72.4 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
3 rows in set (0.00 sec)

Admin>

```

* 在后端MYSQL创建业务使用的账户以及ProxySQL监控使用的账户，这里以app和monitor为例子,任意节点执行

```bash
[root@localhost][mysql]> grant all on *.* to app@'%' identified by 'app';    //实际应用程序使用的账户
Query OK, 0 rows affected, 1 warning (0.00 sec)

[root@localhost][mysql]> GRANT SELECT ON `sys`.* TO 'monitor'@'%'  identified by 'monitor';   //proxysql监控MGR集群使用的账户
Query OK, 0 rows affected, 1 warning (0.00 sec)

[root@localhost][mysql]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

[root@localhost][mysql]> show grants for app;
+------------------------------------------+
| Grants for app@%                         |
+------------------------------------------+
| GRANT ALL PRIVILEGES ON *.* TO 'app'@'%' |
+------------------------------------------+
1 row in set (0.00 sec)

[root@localhost][mysql]>
[root@localhost][mysql]> show grants for monitor;
+------------------------------------------+
| Grants for monitor@%                     |
+------------------------------------------+
| GRANT USAGE ON *.* TO 'monitor'@'%'      |
| GRANT SELECT ON `sys`.* TO 'monitor'@'%' |
+------------------------------------------+
2 rows in set (0.00 sec)

[root@localhost][mysql]>
```

* 配置proxysql,配置监控账户、业务读写账户

```bash
Admin>
Admin>set mysql-monitor_username = 'monitor';               //配置监控账户
Query OK, 1 row affected (0.00 sec)

Admin>set mysql-monitor_password = 'monitor';
Query OK, 1 row affected (0.00 sec)

Admin>load mysql variables to runtime;
Query OK, 0 rows affected (0.01 sec)

Admin>save mysql variables to disk;
Query OK, 143 rows affected (0.00 sec)

Admin>select @@mysql-monitor_username ;
+--------------------------+
| @@mysql-monitor_username |
+--------------------------+
| monitor                  |
+--------------------------+
1 row in set (0.00 sec)

Admin>select @@mysql-monitor_password ;
+--------------------------+
| @@mysql-monitor_password |
+--------------------------+
| monitor                  |
+--------------------------+
1 row in set (0.00 sec)

Admin>insert into mysql_users (username,password,active,default_hostgroup) values ('app','app',1,10);    //配置实际业务读写账户
Query OK, 1 row affected (0.00 sec)

Admin>load mysql users to runtime;
Query OK, 0 rows affected (0.00 sec)

Admin>save mysql users to disk; 
Query OK, 0 rows affected (0.01 se

Admin>select username,password,active,default_hostgroup from runtime_mysql_users;
+----------+-------------------------------------------+--------+-------------------+
| username | password                                  | active | default_hostgroup |
+----------+-------------------------------------------+--------+-------------------+
| app      | *5BCB3E6AC345B435C7C2E6B7949A04CE6F6563D3 | 1      | 10                |
| app      | *5BCB3E6AC345B435C7C2E6B7949A04CE6F6563D3 | 1      | 10                |
+----------+-------------------------------------------+--------+-------------------+
2 rows in set (0.00 sec)

Admin>
```


* 验证proxysql代理能力

```bash
Admin>select * from stats_mysql_query_digest \G                 //可以看到默认都分配到了10组（写组），任何请求，这是因为我们定义用户的规则是这样的
*************************** 1. row ***************************
        hostgroup: 10
       schemaname: zltest01
         username: app
   client_address: 
           digest: 0xB1310F34FA3FF439
      digest_text: select @@server_uuid
       count_star: 1
       first_seen: 1588237206
        last_seen: 1588237206
         sum_time: 1319
         min_time: 1319
         max_time: 1319
sum_rows_affected: 0
    sum_rows_sent: 1
*************************** 2. row ***************************
        hostgroup: 10
       schemaname: zltest01
         username: app
   client_address: 
           digest: 0x99531AEFF718C501
      digest_text: show tables
       count_star: 3
       first_seen: 1588237128
        last_seen: 1588237191
         sum_time: 4058
         min_time: 900
         max_time: 1837
sum_rows_affected: 0
    sum_rows_sent: 1
*************************** 3. row ***************************
        hostgroup: 10
       schemaname: zltest01
         username: app
   client_address: 
           digest: 0x02033E45904D3DF0
      digest_text: show databases
       count_star: 1
       first_seen: 1588237128
        last_seen: 1588237128
         sum_time: 1514
         min_time: 1514
         max_time: 1514
sum_rows_affected: 0
    sum_rows_sent: 9
*************************** 4. row ***************************
        hostgroup: 10
       schemaname: information_schema
         username: app
   client_address: 
           digest: 0x620B328FE9D6D71A
      digest_text: SELECT DATABASE()
       count_star: 1
       first_seen: 1588237128
        last_seen: 1588237128
         sum_time: 1012
         min_time: 1012
         max_time: 1012
sum_rows_affected: 0
    sum_rows_sent: 1
*************************** 5. row ***************************
        hostgroup: 10
       schemaname: zltest01
         username: app
   client_address: 
           digest: 0x3765930C7143F468
      digest_text: select * from t1
       count_star: 1
       first_seen: 1588237195
        last_seen: 1588237195
         sum_time: 2543
         min_time: 2543
         max_time: 2543
sum_rows_affected: 0
    sum_rows_sent: 1
```


* 验证proxysql高可用能力

```bash
Admin>select * from runtime_mysql_servers;     //我们马上关闭这个10组的服务器-10.99.72.6
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| hostgroup_id | hostname   | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment     |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| 10           | 10.99.72.6 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 20           | 10.99.72.5 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 20           | 10.99.72.4 | 3307 | 0         | ONLINE | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
3 rows in set (0.00 sec)

Admin>
Admin>
-----10.99.72.6 已关闭
Admin>
Admin>select * from runtime_mysql_servers;     //可以看到10.99.72.6已经被放置到40离线组中去了，10组里72.5已经顶替上去。
+--------------+------------+------+-----------+---------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| hostgroup_id | hostname   | port | gtid_port | status  | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment     |
+--------------+------------+------+-----------+---------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| 10           | 10.99.72.5 | 3307 | 0         | ONLINE  | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 40           | 10.99.72.6 | 3307 | 0         | SHUNNED | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
| 20           | 10.99.72.4 | 3307 | 0         | ONLINE  | 1      | 0           | 2000            | 0                   | 0       | 0              | MGR-Cluster |
+--------------+------------+------+-----------+---------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
3 rows in set (0.01 sec)

Admin>
```