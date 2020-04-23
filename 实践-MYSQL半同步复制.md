
---
title: 实践-MYSQL半同步复制
date: 2020/04/22
tags: 
  mysql
---
  

## 环境
|OS          |Kernel   |    Software    |
|:-----------|:-----   | :-----------   | 
|CentOS 7.3  |3.10.0   | MySQL / Percona Server     |
 

## 先决条件
* 请自己预先准备好一个普通的异步复制集群

## 注意事项

* 半同步复制需要主和从都安装半同步插件且启用
* 半同步主从复制过程中因网络、系统故障等原因超时会退化成”异步复制“
* 退化成“异步复制”后如果之后网络、系统故障恢复后，将还原到半同步复制的状态

## MYSQL 5.6下半同步复制

|OS          |Kernel   | IP     |   Software    |
|:-----------|:-----   | :---   |:-----------   | 
|CentOS 7.3  |3.10.0   | 10.99.66.152 |5.6.45-log MySQL  |
|CentOS 7.3  |3.10.0   | 10.99.66.153 |5.6.45-log MySQL  |

#### master关键步骤

* 安装半同步复制插件

```bash
mysql> install plugin rpl_semi_sync_master soname 'semisync_master.so';
Query OK, 0 rows affected (0.00 sec)
```

* 启用半同步复制插件

```bash
mysql> set global rpl_semi_sync_master_enabled = ON;
```

* 设置主从复制超时时间，单位是毫秒。如果主从复制超时，即会退化为“异步复制”

```bash
mysql> set global rpl_semi_sync_master_timeout = 3000;
Query OK, 0 rows affected (0.00 sec)
```

* 查看配置

```
mysql> show global variables like '%semi%';
+------------------------------------+-------+
| Variable_name                      | Value |
+------------------------------------+-------+
| rpl_semi_sync_master_enabled       | ON    |    //半同步开启
| rpl_semi_sync_master_timeout       | 3000  |    //超时时间为3秒
| rpl_semi_sync_master_trace_level   | 32    |
| rpl_semi_sync_master_wait_no_slave | ON    |
+------------------------------------+-------+
4 rows in set (0.00 sec)
```

* 写入my.cnf永久生效,在[mysqld]下增加如下

```bash
rpl_semi_sync_master_enabled = 1;
rpl_semi_sync_master_timeout = 3000;
```

#### slave关键步骤

* 安装半同步复制插件

```bash
mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
Query OK, 0 rows affected (0.00 sec)
```

* 启用半同步复制插件

```bash
mysql> set global rpl_semi_sync_slave_enabled = ON;
```

* 重启slave复制，使之生效

```bash
mysql> stop  slave;
mysql> start slave;
```

* 查看配置

```bash
mysql> show global variables like '%semi%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| rpl_semi_sync_slave_enabled     | ON   |
| rpl_semi_sync_slave_trace_level | 32    |
+---------------------------------+-------+
```

#### master验证

```bash
mysql> show global status like '%rpl%';
+--------------------------------------------+--------+
| Variable_name                              | Value  |
+--------------------------------------------+--------+
| Rpl_semi_sync_master_clients               | 1      |    //表示有1个半同步复制节点 
| Rpl_semi_sync_master_net_avg_wait_time     | 1372   |
| Rpl_semi_sync_master_net_wait_time         | 235986 |
| Rpl_semi_sync_master_net_waits             | 172    |
| Rpl_semi_sync_master_no_times              | 3      |
| Rpl_semi_sync_master_no_tx                 | 4      |
| Rpl_semi_sync_master_status                | ON     |    //表示当前集群半同步复制有效 
| Rpl_semi_sync_master_timefunc_failures     | 0      |
| Rpl_semi_sync_master_tx_avg_wait_time      | 1489   |
| Rpl_semi_sync_master_tx_wait_time          | 256251 |
| Rpl_semi_sync_master_tx_waits              | 172    |
| Rpl_semi_sync_master_wait_pos_backtraverse | 0      |
| Rpl_semi_sync_master_wait_sessions         | 0      |
| Rpl_semi_sync_master_yes_tx                | 172    |
+--------------------------------------------+--------+
14 rows in set (0.00 sec)

mysql>
```

## MYSQL 5.7下半同步复制

|OS          |Kernel   | IP     |   Software    |
|:-----------|:-----   | :---   |:-----------   | 
|CentOS 7.3  |3.10.0   | 10.99.66.152 |5.7.21-21-log Percona Server  |
|CentOS 7.3  |3.10.0   | 10.99.66.153 |5.7.21-21-log Percona Server  |


#### master关键步骤

* 安装半同步复制插件

```bash
MySQL> install plugin rpl_semi_sync_master soname 'semisync_master.so';
Query OK, 0 rows affected (0.04 sec)
```

* 启用半同步复制插件

```bash
MySQL> set global rpl_semi_sync_master_enabled  = ON;
Query OK, 0 rows affected (0.00 sec)
```

* 设置主从复制超时时间，单位是毫秒。如果主从复制超时，即会退化为“异步复制”

```bash
mysql> set global rpl_semi_sync_master_timeout = 3000;
Query OK, 0 rows affected (0.00 sec)
```

* 查看配置

```bash
MySQL [(none)]> show variables like '%semi%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | ON         |   //半同步开启
| rpl_semi_sync_master_timeout              | 3000       |  
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |   //5.7多了这个选项，表示无损复制，这个也是默认值。。（AFTER_SYNC和AFTER_COMMIT可选）
+-------------------------------------------+------------+
6 rows in set (0.00 sec)

MySQL [(none)]>
```

* 写入my.cnf永久生效,在[mysqld]下增加如下

```bash
rpl_semi_sync_master_enabled = 1;
rpl_semi_sync_master_timeout = 3000;
```


#### slave关键步骤

* 安装半同步复制插件

```bash
mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
Query OK, 0 rows affected (0.01 sec);
```

* 启用半同步复制插件

```bash
mysql> set global rpl_semi_sync_slave_enabled = ON;
```

* 重启slave复制，使之生效

```bash
mysql> stop  slave;
mysql> start slave;
```

* 查看配置

```bash
mysql> show global variables like '%semi%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| rpl_semi_sync_slave_enabled     | ON   |
| rpl_semi_sync_slave_trace_level | 32    |
+---------------------------------+-------+
```


#### master验证

```bash
MySQL [(none)]> show global status like '%rpl%';
+--------------------------------------------+-------+
| Variable_name                              | Value |
+--------------------------------------------+-------+
| Rpl_semi_sync_master_clients               | 1     |      //表示有1个半同步复制节点 
| Rpl_semi_sync_master_net_avg_wait_time     | 0     |
| Rpl_semi_sync_master_net_wait_time         | 0     |
| Rpl_semi_sync_master_net_waits             | 0     |
| Rpl_semi_sync_master_no_times              | 0     |
| Rpl_semi_sync_master_no_tx                 | 0     |
| Rpl_semi_sync_master_status                | ON    |      //表示当前集群半同步复制有效 
| Rpl_semi_sync_master_timefunc_failures     | 0     |
| Rpl_semi_sync_master_tx_avg_wait_time      | 0     |
| Rpl_semi_sync_master_tx_wait_time          | 0     |
| Rpl_semi_sync_master_tx_waits              | 0     |
| Rpl_semi_sync_master_wait_pos_backtraverse | 0     |
| Rpl_semi_sync_master_wait_sessions         | 0     |
| Rpl_semi_sync_master_yes_tx                | 0     |
+--------------------------------------------+-------+
14 rows in set (0.00 sec)

MySQL [(none)]>
```

## 总结

#### 半同步复制下，复制模式差异？

* 响应从库ack时间点不同，5.6是after_commit即引擎提交之后等待ack，5.7是after_sync即引擎提交之前等待ack最后根据ack结果决定是否提交

#### 半同步复制下，数据一致性差异？

* 主库故障情况下，主从切换，5.6可能导致用户产生“幻读”现象以及主从数据不一致（从丢失了数据）
* 主库故障情况下，主从切换，5.7可能导致用户不会产生“幻读”现象以及主从数据不一致（从增加了数据）

#### 半同步复制下，复制性能差异？

* 5.6使用单线程dump线程来实现binlog日志的传送以及接收从库slave ack响应
* 5.7使用多线程来实现binlog日志的传送以及接收从库slave ack响应，各自有1个线程实现

 
 

