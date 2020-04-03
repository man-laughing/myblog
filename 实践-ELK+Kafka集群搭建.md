
---
title: 实践-ELK+Kafka集群搭建
date: 2020/04/03
tags: 
   ELK
---
  

## 环境
|OS          |Kernel   |IP| Role|Version|
|:-----------|:--------|:-------------|:----|:----|
|CentOS 7.3  |3.10.0   |10.122.132.202 |Elasticsearch,Kafka,zookeeper |elastic6.8/kafka2.12,zk3.14|
|CentOS 7.3  |3.10.0   |10.122.132.203 |Elasticsearch,Kafka,zookeeper |elastic6.8/kafka2.12,zk3.14|
|CentOS 7.3  |3.10.0   |10.122.132.204 |Elasticsearch,Kafka,zookeeper |elastic6.8/kafka2.12,zk3.14|
|CentOS 7.3  |3.10.0   |10.122.132.205 |Logstash,Kibana               |logstash6.8/Kibana6.8|
  
 
## 先决条件
* 所有节点需要安装JAVA运行环境openjdk或oracle jdk
* kafka集群需要依赖zookeeper

## Elasticsearch 

#### 安装依赖JDK

* 在所有Elasticsearch节点安装JDK

```bash
[root@jpoldocker-shqstst-2 ~]# yum install openjdk
```

* 验证JDK

```bash
[root@jpoldocker-shqstst-2 ~]# java -version
openjdk version "12.0.1" 2019-04-16
OpenJDK Runtime Environment (build 12.0.1+12)
OpenJDK 64-Bit Server VM (build 12.0.1+12, mixed mode, sharing)
```

#### 安装Elasticsearch

* 安装Elasticsearch官方repo文件

```bash
[root@jpoldocker-shqstst-1 bin]# cat > /etc/yum.repos.d/elasticsearch.repo << EOF
> [elasticsearch-6.x]
> name=Elasticsearch repository for 6.x packages
> baseurl=https://artifacts.elastic.co/packages/6.x/yum
> gpgcheck=1
> gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
> enabled=1
> autorefresh=1
> type=rpm-md
> EOF
```

NOTE： 该repo仓库文件可以用来安装ELK三个组件～

* 安装Elasticsearch

```bash
[root@jpoldocker-shqstst-1 bin]# yum install elasticsearch --disablerepo=base
```

* 编辑配置文件

```bash
[root@jpoldocker-shqstst-1 elasticsearch]# egrep -v "(^#|^$)"  elasticsearch.yml
cluster.name: leo_escluster  //可以自定义命名
node.name: es-node-1
node.master: true            //本节点可以可以成为主节点
node.data:   true            //本节点可以可以成为数据节点
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.zen.ping.unicast.hosts: ["10.122.132.202","10.122.132.203","10.122.132.204"]       //自动发现的集群地址
discovery.zen.minimum_master_nodes: 2                      //集群内最少的主节点是2可以正常工作
gateway.recover_after_nodes: 2
```

* 启动服务elasticsearch

```bash
[root@jpoldocker-shqstst-1 elasticsearch]# systemctl start elasticsearch.service
```
 

## Kibana 

* 直接yum安装kibanan，因为之前已安装了elastic repo文件了。。

```bash
[root@jpoldocker-shqstst-4 ~]# yum install kibana
```

NOTE: 该组件是主要做数据展示和查询用的，所以在任何机器上安装都可以，只要能正常连接Elasticsearch即可～

* 编辑配置文件

```bash
[root@jpoldocker-shqstst-4 ~]# egrep -v "(^$|^#)"  /etc/kibana/kibana.yml
server.port: 5601
server.host: "10.122.132.202" 
server.name: "leo_kibana"
elasticsearch.hosts: ["http://10.122.132.202:9200"]  //这里指定ES集群的任意地址都可以
logging.dest: /tmp/kibana.log
```

* 启动服务Kibana

```bash
[root@jpoldocker-shqstst-4 ~]# systemctl  start kibana
```
#### 增强elasticserach安全
*  通过x-pack为elasticsearch集群添加安全验证-第一步：激活License

###### 激活License为试用状态，在 Kibana 中访问 Management -> Elasticsearch -> License Management，点击右侧的升级 License 按钮，可以免费试用 30 天的高级 License，升级完成之后页面会显示如下：

![avatar](https://raw.githubusercontent.com/man-laughing/blogphoto/master/res/elk-01.jpg)

*  通过x-pack为elasticsearch集群添加安全验证-第二步：为ES集群设置账户密码

```bash
[root@jpoldocker-shqstst-4 ~]# /usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive //在任意ES集群节点执行均可
```
NOTE: 这一步根据交互默认会设置elastic、kibana、logstash_system 等几个账户，我这里设置账户密码都一样，为了测试方便。

* 编辑elasticsearch配置文件使安全验证生效

```bash
[root@jpoldocker-shqstst-1 elasticsearch]# grep xpack /etc/elasticsearch/elasticsearch.yml
xpack.security.enabled: true  //确保这个配置在你的配置中
```

* 重新启动服务Elasticsearch

```bash
[root@jpoldocker-shqstst-4 ~]# systemctl  restart elasticsearch
```

#### 开启 memory lock内存锁定增强elasticsearch性能

* 配置elasticsearch启动服务脚本对内存资源进行锁定

```bash
[root@jpoldocker-shqstst-1 elasticsearch]# egrep -v "(^$|^#)" /usr/lib/systemd/system/elasticsearch.service
[Unit]
Description=Elasticsearch
Documentation=http://www.elastic.co
Wants=network-online.target
After=network-online.target
[Service]
RuntimeDirectory=elasticsearch
PrivateTmp=true
Environment=ES_HOME=/usr/share/elasticsearch
Environment=ES_PATH_CONF=/etc/elasticsearch
Environment=PID_DIR=/var/run/elasticsearch
EnvironmentFile=-/etc/sysconfig/elasticsearch
WorkingDirectory=/usr/share/elasticsearch
User=elasticsearch
Group=elasticsearch
ExecStart=/usr/share/elasticsearch/bin/elasticsearch -p ${PID_DIR}/elasticsearch.pid --quiet
StandardOutput=journal
StandardError=inherit
LimitNOFILE=65535         //确保此行配置
LimitNPROC=32000          //确保此行配置
LimitAS=infinity
LimitFSIZE=infinity
LimitMEMLOCK=infinity     //确保此行配置
TimeoutStopSec=0
KillSignal=SIGTERM
KillMode=process
SendSIGKILL=no
SuccessExitStatus=143
[Install]
WantedBy=multi-user.target
```

* 编辑elasticsearch配置文件使内存锁定在启动时生效

```bash
[root@jpoldocker-shqstst-1 elasticsearch]# grep mem /etc/elasticsearch/elasticsearch.yml
# Lock the memory on startup:
bootstrap.memory_lock: true       //确保这个配置在你的配置中
```

* 重新启动服务Elasticsearch

```bash
[root@jpoldocker-shqstst-4 ~]# systemctl  restart elasticsearch
```

## Kafka

#### 安装依赖zookeeper集群

* 下载、配置、启动服务

```bash
[root@jpoldocker-shqstst-1 tmp]# wget -S "https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz"
[root@jpoldocker-shqstst-1 tmp]# tar xf zookeeper-3.4.14.tar.gz
[root@jpoldocker-shqstst-1 tmp]# cd zookeeper-3.4.14
[root@jpoldocker-shqstst-1 tmp]# mkdir    /tmp/zookeeper-3.4.14/data
[root@jpoldocker-shqstst-1 tmp]# echo 1 > /tmp/zookeeper-3.4.14/data/myid   //这里zookeeper也是三个节点，所以每台节点的myid值也不同，分别是1，2，3 这里不过多展示
[root@jpoldocker-shqstst-1 tmp]# mkdir    /tmp/zookeeper-3.4.14/log
[root@jpoldocker-shqstst-1 tmp]# cat conf/zoo.cfg    //这是标准的配置文件
clientPort=2181
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper-3.4.14/data
dataLogDir=/tmp/zookeeper-3.4.14/log
server.1=10.122.132.202:2888:3888
server.2=10.122.132.203:2888:3888
server.3=10.122.132.204:2888:3888
[root@jpoldocker-shqstst-1 zookeeper-3.4.14]# bin/zkServer.sh  start  //启动服务
```

 
#### 安装kafka集群 

* 下载

```bash
[root@jpoldocker-shqstst-1 tmp]# wget -S https://www-eu.apache.org/dist/kafka/2.3.0/kafka_2.12-2.3.0.tgz
[root@jpoldocker-shqstst-1 tmp]# tar xf kafka_2.12-2.3.0.tgz
[root@jpoldocker-shqstst-1 tmp]# cd kafka_2.12-2.3.0
```

* 编辑配置文件

```bash
[root@jpoldocker-shqstst-1 kafka_2.12-2.3.0]# egrep -v "(^$|^#)" /tmp/kafka_2.12-2.3.0/config/server.properties
broker.id=0                                            //其他kafka节点修改成自己的，该ID不能重复
advertised.listeners=PLAINTEXT://10.122.132.202:9092   //其他kafka节点修改成自己的
num.network.threads=4
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/tmp/kafka-2.12/log
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=10.122.132.202:2181,10.122.132.203:2181,10.122.132.204:2181   //这里填写zookeeper集群的地址
zookeeper.connection.timeout.ms=6000  
group.initial.rebalance.delay.ms=0
```


* 启动服务

```bash
[root@jpoldocker-shqstst-1 bin]# cd /tmp/kafka_2.12-2.3.0/bin
[root@jpoldocker-shqstst-1 bin]# ./kafka-server-start.sh  -daemon ../config/server.properties
```

* 创建测试topic

```bash
[root@jpoldocker-shqstst-1 bin]# ./kafka-topics.sh --zookeeper 10.122.132.202:2181,10.122.132.203:2181,10.122.132.204:2181 --create test  --partitions 3  --replication-factor 3
[root@jpoldocker-shqstst-1 bin]# ./kafka-topics.sh --zookeeper 10.122.132.202:2181,10.122.132.203:2181,10.122.132.204:2181 --list
__consumer_offsets
routerswitch
test
[root@jpoldocker-shqstst-1 bin]#
```

## Logstash

* 使用yum来安装

```bash
[root@jpoldocker-shqstst-4 ~]# yum install logstash
```

* 编辑logstash配置文件

```bash
[root@jpoldocker-shqstst-4 conf.d]# cat /etc/logstash/conf.d/zltest.conf
input {
   kafka {
        bootstrap_servers => ["10.122.132.202:9092,10.122.132.203:9092,10.122.132.204:9092"]
                   topics => ["test"]   //这里填写刚才Kafka配置的topic
                    codec => json {   charset => "UTF-8"  }
                     type => "test"


   }
}

filter {

if [type] == "test" {

grok {
        patterns_dir => "/etc/logstash/patterns"  //创建自定义patterns
        match => {
	"message" => "%{IP:nginxip}\t%{HTTPDATE:logdate}\t%{HOSTNAME:phost}\t\"(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpVersion})?|%{DATA:rawRequest})\"\t%{NUMBER:responseCode}\t(?:%{NUMBER:bodySize}|-)\t(?:%{QS:referrer}|-)\t(?:%{QS:agent}|-)\t%{NUMBER:requestTime}\t(?:%{DATA:contentType}|-)\t(?:%{URIHOST1:upstreamAddr}|-)\t(?:%{NNUMBER:upstreamCode}|-)\t(?:%{NNUMBER:upstreamResponseTime}|-)\t\"(?:%{FORWARDEDIPTYPE:x_forwarded_for_ip}|-)\"(?:%{OTHER:other}|-)"
        }
	remove_field => [ "message" ]
    }

    date {
        match => [ "logdate", "dd/MMM/YYYY:HH:mm:ss Z" ]
	target => "@timestamp"
    }

    useragent {
        source => "agent"
        target => "ua"
    }

    mutate{
        split=>["x_forwarded_for_ip" ,","]
        add_field => ["clientip", "%{x_forwarded_for_ip[0]}"]
	remove_field => [ "logdate","beat.name","beat.hostname"]
	convert => { "clientip" => "string" }
    }

    #geoip {
    #    source => "clientip"
    #    database => "/opt/app/logstash/map/GeoLite2-City.mmdb"
    #}

    mutate {
        convert => { "httpVersion" => "integer" }
    }

    mutate {
        convert => { "responseCode" => "integer" }
    }

    mutate {
        convert => { "upstreamCode" => "integer" }
    }

    mutate {
        convert => { "bodySize" => "integer" }
    }

    mutate {
        convert => { "requestTime" => "float" }
    }

    mutate {
        convert => { "responseTime" => "float" }
    }

    mutate {
        convert => { "upstreamResponseTime" => "float" }
    }
    mutate {
        convert => { "OTHER" => "integer" }
    }
 }

}


output {
    elasticsearch {
        hosts    => [ "10.122.132.202:9200","10.122.132.203:9200","10.122.132.204:9200" ]
        user     => elastic
        password => elastic
        index    => "logstash-%{[fields][log_topics]}-%{+YYYY.MM.dd}"
    }
}
```
   
* 创建自定义patterns

```bash
[root@jpoldocker-shqstst-4 conf.d]# ll /etc/logstash/patterns
总用量 4
-rw-r--r--. 1 root root 1033 4月   1 10:10 elk.patterns
您在 /var/spool/mail/root 中有邮件
[root@jpoldocker-shqstst-4 conf.d]# cat /etc/logstash/patterns/elk.patterns
DATE_FESB %{YEAR}-%{MONTHNUM}-%{MONTHDAY}
DATE_JAVA %{DATE_FESB}[T ]%{TIME}
DATE_NAXSI %{YEAR}/%{MONTHNUM}/%{MONTHDAY}[T ]%{TIME}
DATE_AUDIT %{YEAR}%{MONTHNUM}%{MONTHDAY}[T ]%{TIME}
DATA_STRINGS [a-zA-Z0-9_-]+|(.*?)
User_Name [a-zA-Z0-9_-]+|(.*?)
Action_Status [CONNECT|FAILED_CONNECT|DISCONNECT]+
CUT_LINE (?:[+|=]){3}
THREADNAME [a-zA-Z0-9_.+-=:()]+
JAVACLASS [a-zA-Z0-9_.,-=:\[\]/]+
REQUESTID [a-zA-Z0-9_.+-=:()]+
CONTENTTYPE [a-zA-Z0-9=+\-;/ ]+
FORWARDEDIPTYPE [0-9\,\-\./ ]+
STRINGS [a-zA-Z.]+
UNUMBER ((?<![0-9.+-])(?>[+-]?(?:(?:[0-9]+(?:\.[0-9]+)?)|(?:\.[0-9]+)))(,\s(?<![0-9.+-])(?>[+-]?(?:(?:[0-9]+(?:\.[0-9]+)?)|(?:\.[0-9]+))))?)|-
NNUMBER [0-9\,\-\./ ]+
UHOST %{IPORHOST}(:\b(?:[1-9][0-9]*)\b)?(,\s%{IPORHOST}(:\b(?:[1-9][0-9]*)\b)?)?
HostName \b(?:[0-9A-Za-z][0-9A-Za-z-]{0,62})(?:\-(?:[0-9A-Za-z][0-9A-Za-z-]{0,62}))*(\-?|\b)
URIHOST1 [a-zA-Z0-9_.,-=:\[\]/ ]+
Any_string \"(.*?)\"|(.*?)|''|""|(.*)?
OTHER .*
STATUS [ok|fail|sucess]+
MAILNAME [a-zA-Z\.0-9\-_]+@[a-zA-Z0-9\.]+
MAILNAME2 [a-zA-Z\@\.0-9\-_]+
RESULT [A-Z_]+
```   

* 启动logstash服务

```bash
[root@jpoldocker-shqstst-4 conf.d]# /usr/share/logstash/bin/logstash -r -f /etc/logstash/conf.d/zltest.conf   &> app.log &
```


## Filebeat 

* 安装

```bash
[root@netscalelog-shylf-2 filebeat]# yum install filebeat
```

* 编辑配置文件

```bash
[root@netscalelog-shylf-2 filebeat]# cat filebeat.yml
filebeat.prospectors:
- type: log
  enabled: true
  paths:
    - /tmp/ess.access.log
  fields:
    log_topics: routerswitch

output.kafka:
  enabled: true
  hosts: ["10.122.132.202:9092","10.122.132.203:9092","10.122.132.204:9092"]
  topic: '%{[fields][log_topics]}'
  partition.hash:
  reachable_only: true
  compression: gzip
  max_message_bytes: 1000000
  required_acks: 1
[root@netscalelog-shylf-2 filebeat]#
```

* 启动服务

```bash
[root@netscalelog-shylf-2 filebeat]#  systemctl  start filebeat
```

