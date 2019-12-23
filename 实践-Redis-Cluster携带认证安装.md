
---
title: 实践-Redis-Cluster携带认证安装
date: 2019/12/23
tags: 
   redis
---
  

## 环境
|OS          |Kernel   | Redis Version| Ruby Version | Gem Version|
|:-----------|:--------|:-------------| :------------|:-----------|
|CentOS 7.3  |3.10.0   | 3.2.6        | 2.4.4p296    | 2.6.14.1   |

## 须知
* 本实验在单机CentOS 7上执行,采用三个分片（shard），每个分片为一主一从模式，一共6个实例

## 先决条件
* 请确保你已安装Ruby环境
* 请确保你已安装Ruby包管理器Gem实用程序

## 实际案例 


#### 初始化Redis-Cluster

* 安装Ruby及各种依赖

```bash
[root@localhost ~]# yum install ruby //我这里是自己打的RPM包，携带了gem程序
[root@localhost ~]# 
[root@localhost ~]# ruby  --version
ruby 2.4.4p296 (2018-03-28 revision 63013) [x86_64-linux]
[root@localhost ~]# 
[root@localhost ~]# gem --version
2.6.14.1
[root@localhost ~]# gem install redis  //管理脚本redis-trib.rb依赖Redis库，必须安装
Successfully installed redis-4.1.3
Parsing documentation for redis-4.1.3
Done installing documentation for redis after 1 seconds
1 gem installed
[root@localhost ~]#
```

* 安装Redis

```bash
[root@localhost ~]#  yum install redis //我这里是自己打的RPM包，包里携带了redis-trib.rb脚本，默认redis3.x的源代码包里有该脚本，也可以自行获取
```

* 准备配置文件

```bash
[root@localhost zltest]#  pwd
/opt/app/redis/zltest
[root@localhost zltest]# 
[root@localhost zltest]# tree  //目录结构如下
.
├── conf
│   ├── redis-6381.conf
│   ├── redis-6382.conf
│   ├── redis-6383.conf
│   ├── redis-6384.conf
│   ├── redis-6385.conf
│   └── redis-6386.conf
├── data
│   ├── redis-6381
│   ├── redis-6382
│   ├── redis-6383
│   ├── redis-6384
│   ├── redis-6385
│   └── redis-6386
├── log
└── pid

10 directories, 6 files
[root@localhost zltest]#
[root@localhost zltest]# cat conf/redis-6381.conf  //其余的配置文件以此为准，修改下端口即可（这里不浪费篇幅展示了）
daemonize yes
pidfile "/opt/app/redis/zltest/pid/redis-6381.pid"
port 6381
timeout 300
loglevel notice
logfile "/opt/app/redis/zltest/log/redis-6381.log"
databases 16
dbfilename "dump-6381.rdb"
slave-serve-stale-data no
slave-read-only yes
slave-priority 100
bind 10.116.71.209 127.0.0.1
maxmemory 500000kb
maxmemory-policy allkeys-lru
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-entries 512
list-max-ziplist-value 64
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
dir "/opt/app/install/redis-3.2.6/zltest/data/redis-6381"
no-appendfsync-on-rewrite yes
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
auto-aof-rewrite-min-size 512mb
auto-aof-rewrite-percentage 100
aof-load-truncated yes
aof-rewrite-incremental-fsync yes
cluster-enabled yes
cluster-config-file "redis-6381-nodes.conf"
cluster-node-timeout 5000
```

* 启动Redis-Server

```bash
[root@localhost zltest]# redis-server  --version
Redis server v=3.2.6 sha=00000000:0 malloc=libc bits=64 build=e33083d7b0950e51
[root@localhost zltest]#
[root@localhost zltest]# redis-server conf/redis-6381.conf  //没有任何认证信息
[root@localhost zltest]# redis-server conf/redis-6382.conf  //没有任何认证信息
[root@localhost zltest]# redis-server conf/redis-6383.conf  //没有任何认证信息
[root@localhost zltest]# redis-server conf/redis-6384.conf  //没有任何认证信息
[root@localhost zltest]# redis-server conf/redis-6385.conf  //没有任何认证信息
[root@localhost zltest]# redis-server conf/redis-6386.conf  //没有任何认证信息
[root@localhost zltest]#
[root@localhost zltest]# ps aux |grep redis
root     18905  0.0  0.0 133868  2392 ?        Ssl  20:30   0:00 redis-server 10.116.71.209:6381 [cluster]
root     18918  0.0  0.0 133868  2392 ?        Ssl  20:30   0:00 redis-server 10.116.71.209:6382 [cluster]
root     19021  0.0  0.0 133868  2392 ?        Ssl  20:30   0:00 redis-server 10.116.71.209:6383 [cluster]
root     19025  0.0  0.0 133868  2396 ?        Ssl  20:30   0:00 redis-server 10.116.71.209:6384 [cluster]
root     19029  0.0  0.0 133868  2392 ?        Ssl  20:30   0:00 redis-server 10.116.71.209:6385 [cluster]
root     19033  0.0  0.0 133868  2392 ?        Ssl  20:30   0:00 redis-server 10.116.71.209:6386 [cluster]
root     19221  0.0  0.0 112712   972 pts/2    R+   20:31   0:00 grep --color=auto redis
[root@localhost zltest]#
```

* 通过 redis-trib.rb 管理脚本创建Redis-Cluster集群

```bash
[root@localhost ~]# redis-trib.rb  create --replicas 1 10.116.71.209:6381 10.116.71.209:6382 10.116.71.209:6383 10.116.71.209:6384 10.116.71.209:6385 10.116.71.209:6386
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
10.116.71.209:6381
10.116.71.209:6382
10.116.71.209:6383
Adding replica 10.116.71.209:6384 to 10.116.71.209:6381
Adding replica 10.116.71.209:6385 to 10.116.71.209:6382
Adding replica 10.116.71.209:6386 to 10.116.71.209:6383
M: cf582d52ccb471b5f27a80c208823f5b6f2745b1 10.116.71.209:6381
   slots:0-5460 (5461 slots) master
M: 897ed51483fd105a6bcb881f0a72de1cc6ff694a 10.116.71.209:6382
   slots:5461-10922 (5462 slots) master
M: ea6baa60a9aa784454c2d69e662a686ae2b53e7f 10.116.71.209:6383
   slots:10923-16383 (5461 slots) master
S: d331353d2b36b1e938bd665f403c2da8001ba78d 10.116.71.209:6384
   replicates cf582d52ccb471b5f27a80c208823f5b6f2745b1
S: 71a153857796f008f8be17793afbe63586e9c91c 10.116.71.209:6385
   replicates 897ed51483fd105a6bcb881f0a72de1cc6ff694a
S: d91827aaf79b902d2c0f25af1eba01ee126c7de8 10.116.71.209:6386
   replicates ea6baa60a9aa784454c2d69e662a686ae2b53e7f
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join...
[ERR] Unable to load info for node 10.116.71.209:6384
[ERR] Unable to load info for node 10.116.71.209:6386
[ERR] Unable to load info for node 10.116.71.209:6385
>>> Performing Cluster Check (using node 10.116.71.209:6381)
M: cf582d52ccb471b5f27a80c208823f5b6f2745b1 10.116.71.209:6381
   slots:0-5460 (5461 slots) master
   0 additional replica(s)
M: ea6baa60a9aa784454c2d69e662a686ae2b53e7f 10.116.71.209:6383
   slots:10923-16383 (5461 slots) master
   0 additional replica(s)
M: 897ed51483fd105a6bcb881f0a72de1cc6ff694a 10.116.71.209:6382
   slots:5461-10922 (5462 slots) master
   0 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
[root@localhost ~]#
```

* 检查Redis-Cluster集群是否配置完成

```bash
[root@localhost  ~]# redis-trib.rb  check 127.0.0.1:6381    //可以看到三个分片都是一主一从的结构，比较均衡
>>> Performing Cluster Check (using node 127.0.0.1:6381)
M: cf582d52ccb471b5f27a80c208823f5b6f2745b1 127.0.0.1:6381
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
M: ea6baa60a9aa784454c2d69e662a686ae2b53e7f 10.116.71.209:6383
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: d331353d2b36b1e938bd665f403c2da8001ba78d 10.116.71.209:6384
   slots: (0 slots) slave
   replicates cf582d52ccb471b5f27a80c208823f5b6f2745b1
M: 897ed51483fd105a6bcb881f0a72de1cc6ff694a 10.116.71.209:6382
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: d91827aaf79b902d2c0f25af1eba01ee126c7de8 10.116.71.209:6386
   slots: (0 slots) slave
   replicates ea6baa60a9aa784454c2d69e662a686ae2b53e7f
S: 71a153857796f008f8be17793afbe63586e9c91c 10.116.71.209:6385
   slots: (0 slots) slave
   replicates 897ed51483fd105a6bcb881f0a72de1cc6ff694a
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
[root@localhost  ~]#
```

* 检查Redis-Cluster集群正常功能是否完整(模拟KEY的写入)

```bash
[root@localhost ~]# redis-cli -c -p 6381
127.0.0.1:6381> set testkey testvalue      //这个key根据哈希规则后最终写入到了6381实例的主从集群
OK
127.0.0.1:6381> set testkey2 testvalue2    //这个key根据哈希规则后最终写入到了6383实例的主从集群
-> Redirected to slot [14758] located at 10.116.71.209:6383     
OK
10.116.71.209:6383> set testkey3 testvalue3
-> Redirected to slot [10631] located at 10.116.71.209:6382   //这个key根据哈希规则后最终写入到了6382实例的主从集群

OK
10.116.71.209:6382>
```

* 检查Redis-Cluster的哈希key的分布情况

```bash
[root@localhost ~]# redis-trib.rb  info 127.0.0.1:6381       //可以看到每个分片都写入了一个KEY
127.0.0.1:6381 (cf582d52...) -> 1 keys | 5461 slots | 1 slaves.
10.116.71.209:6383 (ea6baa60...) -> 1 keys | 5461 slots | 1 slaves.
10.116.71.209:6382 (897ed514...) -> 1 keys | 5462 slots | 1 slaves.
[OK] 3 keys in 3 masters.
0.00 keys per slot on average.
[root@localhost ~]#
```

#### 为Redis-Cluster添加安全验证

* 设置安全验证（添加Redis密码）

```bash
[root@localhost ~]# redis-cli -c -p 6381    //为第一个分片的主节点添加安全验证
127.0.0.1:6381> 
127.0.0.1:6381> CONFIG GET masterauth
1) "masterauth"
2) ""
127.0.0.1:6381> CONFIG GET requirepass
1) "requirepass"
2) ""
127.0.0.1:6381>
127.0.0.1:6381> CONFIG SET  masterauth testpass
OK
127.0.0.1:6381> CONFIG SET  requirepass testpass
OK
127.0.0.1:6381> CONFIG rewrite  //这里保存配置失败，因为上面设置了密码，立刻就生效了
(error) NOAUTH Authentication required.
127.0.0.1:6381> AUTH testpass
OK
127.0.0.1:6381> CONFIG rewrite   //保存配置 
OK
127.0.0.1:6381>


[root@localhost ~]# redis-cli -c -p 6382  //为第二个分片的主节点添加安全验证
127.0.0.1:6382> CONFIG GET masterauth  
1) "masterauth"
2) ""
127.0.0.1:6382> CONFIG GET requirepass
1) "requirepass"
2) ""
127.0.0.1:6382> CONFIG SET  masterauth  testpass
OK
127.0.0.1:6382> CONFIG SET  requirepass  testpass
OK
127.0.0.1:6382> CONFIG rewrite   //这里保存配置失败，因为上面设置了密码，立刻就生效了
(error) NOAUTH Authentication required.
127.0.0.1:6382>  AUTH testpass
OK
127.0.0.1:6382> CONFIG rewrite   //保存配置 
OK
127.0.0.1:6382>


[root@localhost ~]# redis-cli -c -p 6383    //为第三个分片的主节点添加安全验证
127.0.0.1:6383>
127.0.0.1:6383>  CONFIG GET masterauth
1) "masterauth"
2) ""
127.0.0.1:6383>  CONFIG GET requirepass
1) "requirepass"
2) ""
127.0.0.1:6383> CONFIG SET masterauth testpass
OK
127.0.0.1:6383> CONFIG SET requirepass  testpass
OK
127.0.0.1:6383> CONFIG rewrite    //这里保存配置失败，因为上面设置了密码，立刻就生效了
(error) NOAUTH Authentication required.
127.0.0.1:6383>
127.0.0.1:6383> AUTH testpass
OK
127.0.0.1:6383> CONFIG rewrite    //保存配置 
OK
127.0.0.1:6383>


[root@localhost conf]# redis-cli -c -p 6384   //为第一个分片的从节点添加安全验证
127.0.0.1:6384> CONFIG GET
(error) ERR Wrong number of arguments for CONFIG GET
127.0.0.1:6384> CONFIG GET  masterauth
1) "masterauth"
2) ""
127.0.0.1:6384> CONFIG SET  masterauth testpass
OK
127.0.0.1:6384> CONFIG SET requirepass  testpass
OK
127.0.0.1:6384> CONFIG rewrite     //这里保存配置失败，因为上面设置了密码，立刻就生效了
(error) NOAUTH Authentication required.
127.0.0.1:6384> AUTH testpass
OK
127.0.0.1:6384> CONFIG rewrite     //保存配置 
OK
127.0.0.1:6384> exit


[root@prometheus-shanghai-2 conf]# redis-cli -c -p 6385    //为第二个分片的从节点添加安全验证
127.0.0.1:6385>
127.0.0.1:6385> CONFIG GET  masterauth
1) "masterauth"
2) ""
127.0.0.1:6385> CONFIG SET  masterauth testpass
OK
127.0.0.1:6385> CONFIG SET requirepass  testpass
OK
127.0.0.1:6385>  CONFIG rewrite     //这里保存配置失败，因为上面设置了密码，立刻就生效了
(error) NOAUTH Authentication required.
127.0.0.1:6385> AUTH testpass
OK
127.0.0.1:6385> CONFIG rewrite      //保存配置 
OK
127.0.0.1:6385>


[root@prometheus-shanghai-2 conf]# redis-cli -c -p 6386   //为第三个分片的从节点添加安全验证
127.0.0.1:6386>
127.0.0.1:6386>  CONFIG GET  masterauth
1) "masterauth"
2) ""
127.0.0.1:6386>  CONFIG SET  masterauth testpass
OK
127.0.0.1:6386>  CONFIG SET requirepass  testpass
OK 
127.0.0.1:6386> CONFIG rewrite      //这里保存配置失败，因为上面设置了密码，立刻就生效了
(error) NOAUTH Authentication required.
127.0.0.1:6386>  AUTH testpass
OK
127.0.0.1:6386>  CONFIG rewrite     //保存配置 
OK
127.0.0.1:6386>
```

* 检查上面的安全验证是否添加成功(是否同步到了配置文件中)

```bash
[root@localhost conf]# pwd
/opt/app/redis/zltest/conf  
[root@localhost conf]# grep testpass *.conf   //这里可以看到全部都生效了，就算重启也依然生效
redis-6381.conf:masterauth "testpass"
redis-6381.conf:requirepass "testpass"
redis-6382.conf:masterauth "testpass"
redis-6382.conf:requirepass "testpass"
redis-6383.conf:masterauth "testpass"
redis-6383.conf:requirepass "testpass"
redis-6384.conf:masterauth "testpass"
redis-6384.conf:requirepass "testpass"
redis-6385.conf:masterauth "testpass"
redis-6385.conf:requirepass "testpass"
redis-6386.conf:masterauth "testpass"
redis-6386.conf:requirepass "testpass"
```

#### 使用管理脚本 redis-trib.rb 查看验证

* 验证

```bash
[root@localhost conf]# redis-trib.rb check 127.0.0.1:6381  //因为我们的集群添加了安全验证，所以这里redis-trib.rb也要添加验证
[ERR] Sorry, can't connect to node 127.0.0.1:6381
[root@localhost conf]#
[root@localhost conf]# find / -name "client.rb"
/opt/app/install/ruby-2.4.4/lib/ruby/gems/2.4.0/gems/xmlrpc-0.2.1/lib/xmlrpc/client.rb
/opt/app/install/ruby-2.4.4/lib/ruby/gems/2.4.0/gems/redis-4.1.3/lib/redis/client.rb   //修改这个文件
[root@localhost conf]# head -n 20 /opt/app/install/ruby-2.4.4/lib/ruby/gems/2.4.0/gems/redis-4.1.3/lib/redis/client.rb
# frozen_string_literal: true

require_relative "errors"
require "socket"
require "cgi"

class Redis
  class Client

    DEFAULTS = {
      :url => lambda { ENV["REDIS_URL"] },
      :scheme => "redis",
      :host => "127.0.0.1",
      :port => 6379,
      :path => nil,
      :timeout => 5.0,
      :password => "testpass",  //修改这个密码为集群实际密码
      :db => 0,
      :driver => nil,
      :id => nil,

[root@localhost conf]# redis-trib.rb check 127.0.0.1:6381   // 好了 :) 
>>> Performing Cluster Check (using node 127.0.0.1:6381)
M: cf582d52ccb471b5f27a80c208823f5b6f2745b1 127.0.0.1:6381
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
M: ea6baa60a9aa784454c2d69e662a686ae2b53e7f 10.116.71.209:6383
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: d331353d2b36b1e938bd665f403c2da8001ba78d 10.116.71.209:6384
   slots: (0 slots) slave
   replicates cf582d52ccb471b5f27a80c208823f5b6f2745b1
M: 897ed51483fd105a6bcb881f0a72de1cc6ff694a 10.116.71.209:6382
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: d91827aaf79b902d2c0f25af1eba01ee126c7de8 10.116.71.209:6386
   slots: (0 slots) slave
   replicates ea6baa60a9aa784454c2d69e662a686ae2b53e7f
S: 71a153857796f008f8be17793afbe63586e9c91c 10.116.71.209:6385
   slots: (0 slots) slave
   replicates 897ed51483fd105a6bcb881f0a72de1cc6ff694a
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
[root@localhost conf]#      
```