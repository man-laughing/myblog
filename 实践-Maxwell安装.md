---
  title: 实践-Maxwell安装
  date: 2021/12/31
  tags: 
     maxwell
---

## 简介
这是Maxwell的守护进程，这是一个读取MySQL binlogs并将行更新写入Kafka，Kinesis，RabbitMQ，Google Cloud Pub / Sub或Redis（Pub / Sub或LPUSH）作为JSON的应用程序。 Maxwell具有较低的操作栏，可生成一致，易于摄取的更新流。它允许您轻松“固定”流处理系统的一些优点，而无需通过整个代码库来添加（不可靠）检测点。常见用例包括ETL，缓存构建/到期，度量收集，搜索索引和服务间通信

## 系统环境
| OS      |  Kernel |IP   |Hostname|
|:----    |:--------|:----|:----|
|CentOS7.6|3.10.0   |10.99.73.251|dbbackup|
 

## 软件环境
| software    |  version | 
|:----    |:--------| 
|JDK| 11.0.13  | 
|Kafka| 2.13  | 
|Maxwell| 1.35 | 

  

## 先决条件

* 安装JDK11 

```bash
# rpm -ivh  jdk-11.0.13_linux-x64_bin.rpm
```

* 安装Kafka && 启动 zookeeper && 启动Kafka

```bash
# nohup bin/zookeeper-server-start.sh config/zookeeper.properties &> boot_zookeeper.log &
# nohup bin/kafka-server-start.sh config/server.properties &> boot_kafka.log &
```
 
* 被监控、采集的MYSQL服务端的配置

```
# cat my.cnf

[mysqld]
server_id=1               //必须设置
log-bin=master         //必须设置
binlog_format=row    //必须设置为ROW
``` 

* 被监控、采集的MYSQL服务端的Maxwell账号权限配置

```
mysql> CREATE USER 'zltest'@'%' IDENTIFIED BY 'zltest';
mysql> GRANT ALL ON maxwell.* TO 'zltest'@'%';
mysql> GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'zltest'@'%';
``` 
 
 

## 安装 
* 下载

```bash
# cd /usr/local/src/
# wget -S "https://github.com/zendesk/maxwell/releases/download/v1.35.5/maxwell-1.35.5.tar.gz"
# ll
total 108
drwxr-xr-x. 2 root root  4096 Dec 29 15:40 bin
-rwxr-xr-x. 1 root root    86 Dec 29 17:24 boot
-rw-r--r--. 1 root root 27202 Jun 23  2021 config.md
-rw-r--r--. 1 root root   257 Dec 31 16:21 config.properties
-rw-r--r--. 1 root root 12095 Dec 29 16:59 config.properties.bak
-rw-r--r--. 1 root root 11970 Jan 28  2021 config.properties.example
-rw-r--r--. 1 root root 10259 Apr 23  2020 kinesis-producer-library.properties.example
drwxr-xr-x. 3 root root  8192 Jul 30 01:41 lib
-rw-r--r--. 1 root root   548 Apr 23  2020 LICENSE
-rw-r--r--. 1 root root   470 Jan 28  2021 log4j2.xml
-rw-r--r--. 1 root root  2652 Dec 31 16:27 maxwell_running.log
-rw-r--r--. 1 root root  3618 Jul 30 01:40 quickstart.md
-rw-r--r--. 1 root root  1429 Jul 30 01:40 README.md
[root@dbbackup maxwell-1.35]#
```

* 编辑maxwell配置文件

```
[root@dbbackup maxwell-1.35]# cat config.properties
log_level=info
producer=kafka            //表示输出到kafka
kafka.bootstrap.servers=10.99.73.251:9092    //输出到Kafka的地址
host=10.99.73.251         //MYSQL的地址      
port=3501                    //MYSQL的端口     
user=zltest                  //MYSQL的账号     
password=zltest           //MYSQL的密码
kafka_topic=maxwell     //kafka的主题名字，默认maxwell  
#output_ddl=true      
#ddl_kafka_topic=maxwell_ddl
kafka.compression.type=snappy
kafka.retries=0
kafka.acks=1
```

* 启动

```
# nohup bin/maxwell --config config.properties  &>  maxwell_running.log  &
```



 
 