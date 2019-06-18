---
 title: 实践-对象存储Swift扩容
 date:  2019/06/18
 tags: 
   分布式存储
---

## 环境
|  OS  |  Kernel | IP     | Version| Role | Note |
|:-----|:--------|:-----|:---------|:-----|:-----|
|CentOS 7.6|3.10.0|10.122.138.231|openstack ocata| proxy/storage |-|
|CentOS 7.6|3.10.0|10.122.138.232|openstack ocata| proxy/storage |-|
|CentOS 7.6|3.10.0|**10.122.138.243**|openstack ocata|storage |新增节点 10G*2 disk|


## 先决条件

NOTE: 以下所有操作在新增节点执行

* 清空iptables规则

```bash
[root@localhost ~]# iptables -F 
[root@localhost ~]# iptables -X
```

* 关闭或禁用selinux

```bash
[root@localhost ~]# setenforce 0
[root@localhost ~]# getenforce    //我这里已经提前禁用
Disabled
```

* 格式化2块10GB磁盘并且挂载

NOTE: 我这里使用vmware虚拟机模拟真实环境，fdisk格式化磁盘过程略

```bash
[root@localhost ~]# mkfs.xfs /dev/sdb1 
[root@localhost ~]# mkfs.xfs /dev/sdc1 
[root@localhost ~]# mkdir -p /data/swiftstorage/data01
[root@localhost ~]# mkdir -p /data/swiftstorage/data02
[root@localhost ~]# mount /dev/sdb1 /data/swiftstorage/data01
[root@localhost ~]# mount /dev/sdc1 /data/swiftstorage/data02
```

* 安装OpenStack Swift服务和rsyncd服务

```bash
[root@localhost ~]# yum install centos-release-openstack-ocata 
[root@localhost ~]# yum install openstack-swift-account openstack-swift-proxy  openstack-swift-container openstack-swift-object python-swift  python2-swiftclient rsync
```


## 扩容

NOTE: 这里选择扩容新增节点这台机器（10.122.138.243），机器上共2块10GB磁盘
 
* 查看集群内扩容前的哈希环RING文件

```bash
[root@swift01 swift]# swift-ring-builder account.builder 
account.builder, build version 5
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 4 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file account.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6002 10.122.138.241:6002 data01 100.00        512    0.00       
            1      1    1 10.122.138.241:6002 10.122.138.241:6002 data02 100.00        512    0.00       
            2      1    2 10.122.138.242:6002 10.122.138.242:6002 data01 100.00        512    0.00       
            3      1    2 10.122.138.242:6002 10.122.138.242:6002 data02 100.00        512    0.00       
[root@swift01 swift]# swift-ring-builder container.builder 
container.builder, build version 5
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 4 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file container.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6001 10.122.138.241:6001 data01 100.00        512    0.00       
            1      1    1 10.122.138.241:6001 10.122.138.241:6001 data02 100.00        512    0.00       
            2      1    2 10.122.138.242:6001 10.122.138.242:6001 data01 100.00        512    0.00       
            3      1    2 10.122.138.242:6001 10.122.138.242:6001 data02 100.00        512    0.00       
[root@swift01 swift]# swift-ring-builder object.builder 
object.builder, build version 5
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 4 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file object.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6000 10.122.138.241:6000 data01 100.00        512    0.00       
            1      1    1 10.122.138.241:6000 10.122.138.241:6000 data02 100.00        512    0.00       
            2      1    2 10.122.138.242:6000 10.122.138.242:6000 data01 100.00        512    0.00       
            3      1    2 10.122.138.242:6000 10.122.138.242:6000 data02 100.00        512    0.00       
[root@swift01 swift]# 
```

* 配置哈希环RING文件

```bash
[root@swift02 swift]# swift-ring-builder account.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6002 --device data01 --weight 100
[root@swift02 swift]# swift-ring-builder account.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6002 --device data02 --weight 100

[root@swift02 swift]# swift-ring-builder container.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6001 --device data01 --weight 100 
[root@swift02 swift]# swift-ring-builder container.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6001 --device data02 --weight 100 

[root@swift02 swift]# swift-ring-builder object.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6000 --device data01 --weight 100
[root@swift02 swift]# swift-ring-builder object.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6000 --device data02 --weight 100

[root@swift02 swift]# swift-ring-builder account.builder    write_ring         //写入RING哈希环，使之生效
[root@swift02 swift]# swift-ring-builder container.builder  write_ring         //写入RING哈希环，使之生效
[root@swift02 swift]# swift-ring-builder object.builder     write_ring         //写入RING哈希环，使之生效

[root@swift02 swift]# swift-ring-builder account.builder   rebalance       //集群分区重均衡
[root@swift02 swift]# swift-ring-builder container.builder rebalance       //集群分区重均衡
[root@swift02 swift]# swift-ring-builder object.builder    rebalance       //集群分区重均衡
 
```

* 分发哈希环RING文件到集群所有节点

```bash
[root@swift02 swift]# scp -r -P 22 *.gz       root@10.122.138.241:/etc/swift/
[root@swift02 swift]# scp -r -P 22 *.gz       root@10.122.138.243:/etc/swift/
[root@swift02 swift]# scp -r -P 22 *.builder  root@10.122.138.241:/etc/swift/
[root@swift02 swift]# scp -r -P 22 *.builder  root@10.122.138.243:/etc/swift/
```

* 确保各个角色server的配置文件正确

```bash
account:
  - /etc/swift/account-server.conf
container:
  - /etc/swift/container-server.conf
object:
  - /etc/swift/object-server.conf
```

* 启动rsync服务（此操作在新增节点执行）

```bash
[root@localhost ~]# systemctl start rsyncd  
```

* 启动swift服务（此操作在新增节点执行）

NOTE：一定确保新增节点的哈希环文件\*.builder和\*.gz文件与集群内其他节点的一致性

```bash
[root@localhost ~]# sh swift.sh start
```


## 验证

* 查看当前哈希环RING状态

```bash
[root@localhost swift]# swift-ring-builder account.builder 
account.builder, build version 11
1024 partitions, 2.000000 replicas, 1 regions, 3 zones, 6 devices, 0.20 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:19:21 remaining)
The overload factor is 0.00% (0.000000)
Ring file account.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6002 10.122.138.241:6002 data01 100.00        342    0.20       
            1      1    1 10.122.138.241:6002 10.122.138.241:6002 data02 100.00        341   -0.10       
            2      1    2 10.122.138.242:6002 10.122.138.242:6002 data01 100.00        342    0.20       
            3      1    2 10.122.138.242:6002 10.122.138.242:6002 data02 100.00        341   -0.10       
            4      1    3 10.122.138.243:6002 10.122.138.243:6002 data01 100.00        341   -0.10       
            5      1    3 10.122.138.243:6002 10.122.138.243:6002 data02 100.00        341   -0.10       
[root@localhost swift]# swift-ring-builder container.builder 
container.builder, build version 8
1024 partitions, 2.000000 replicas, 1 regions, 3 zones, 6 devices, 0.20 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:44:15 remaining)
The overload factor is 0.00% (0.000000)
Ring file container.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6001 10.122.138.241:6001 data01 100.00        342    0.20       
            1      1    1 10.122.138.241:6001 10.122.138.241:6001 data02 100.00        341   -0.10       
            2      1    2 10.122.138.242:6001 10.122.138.242:6001 data01 100.00        342    0.20       
            3      1    2 10.122.138.242:6001 10.122.138.242:6001 data02 100.00        341   -0.10       
            4      1    3 10.122.138.243:6001 10.122.138.243:6001 data01 100.00        341   -0.10       
            5      1    3 10.122.138.243:6001 10.122.138.243:6001 data02 100.00        341   -0.10       
[root@localhost swift]# swift-ring-builder object.builder 
object.builder, build version 8
1024 partitions, 2.000000 replicas, 1 regions, 3 zones, 6 devices, 0.20 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:44:20 remaining)
The overload factor is 0.00% (0.000000)
Ring file object.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6000 10.122.138.241:6000 data01 100.00        342    0.20       
            1      1    1 10.122.138.241:6000 10.122.138.241:6000 data02 100.00        341   -0.10       
            2      1    2 10.122.138.242:6000 10.122.138.242:6000 data01 100.00        342    0.20       
            3      1    2 10.122.138.242:6000 10.122.138.242:6000 data02 100.00        341   -0.10       
            4      1    3 10.122.138.243:6000 10.122.138.243:6000 data01 100.00        341   -0.10       
            5      1    3 10.122.138.243:6000 10.122.138.243:6000 data02 100.00        341   -0.10       
[root@localhost swift]# 
```
 
*  上传一个大文件，这里以2G为例子

```bash
[root@swift01 ~]# swift -A http://10.122.138.242:8080/auth/v1.0 -U admin:admin -K admin upload test02  2G.test.zip
```

* 验证文件分布

方法1：通过文件大小来辨别

```bash
[root@swift02 ~]# find /data/swiftstorage/ -type f -size +1500M -exec ls -lh {} \;
-rw------- 1 swift swift 2.0G Jun 18 19:36 /data/swiftstorage/data01/objects/979/c84/f4d1c4252b8349885721271db7bcfc84/1560857787.11256.data
[root@swift02 ~]# 

[root@localhost ~]# find /data/swiftstorage/ -type f -size +1500M -exec ls -lh {} \;
-rw------- 1 swift swift 2.0G 6月  18 19:36 /data/swiftstorage/data01/objects/979/c84/f4d1c4252b8349885721271db7bcfc84/1560857787.11256.data
[root@localhost ~]# 

```


方法2：通过命令工具来完成

```bash
[root@swift01 ~]#  swift-get-nodes  -a /etc/swift/object.ring.gz  LAUGHING_admin/test02/2G.test.zip

Account  	LAUGHING_admin
Container	test02
Object   	2G.test.zip


Partition	979
Hash     	f4d1c4252b8349885721271db7bcfc84

Server:Port Device	10.122.138.243:6000 data01
Server:Port Device	10.122.138.242:6000 data01
Server:Port Device	10.122.138.241:6000 data01	 [Handoff]
Server:Port Device	10.122.138.242:6000 data02	 [Handoff]
Server:Port Device	10.122.138.243:6000 data02	 [Handoff]
Server:Port Device	10.122.138.241:6000 data02	 [Handoff]


curl -g -I -XHEAD "http://10.122.138.243:6000/data01/979/LAUGHING_admin/test02/2G.test.zip"    //由此说明刚传的文件已经传到了我们新增节点上了，扩容操作有效
curl -g -I -XHEAD "http://10.122.138.242:6000/data01/979/LAUGHING_admin/test02/2G.test.zip"
curl -g -I -XHEAD "http://10.122.138.241:6000/data01/979/LAUGHING_admin/test02/2G.test.zip" # [Handoff]
curl -g -I -XHEAD "http://10.122.138.242:6000/data02/979/LAUGHING_admin/test02/2G.test.zip" # [Handoff]
curl -g -I -XHEAD "http://10.122.138.243:6000/data02/979/LAUGHING_admin/test02/2G.test.zip" # [Handoff]
curl -g -I -XHEAD "http://10.122.138.241:6000/data02/979/LAUGHING_admin/test02/2G.test.zip" # [Handoff]


Use your own device location of servers:
such as "export DEVICE=/srv/node"
ssh 10.122.138.243 "ls -lah ${DEVICE:-/srv/node*}/data01/objects/979/c84/f4d1c4252b8349885721271db7bcfc84"
ssh 10.122.138.242 "ls -lah ${DEVICE:-/srv/node*}/data01/objects/979/c84/f4d1c4252b8349885721271db7bcfc84"
ssh 10.122.138.241 "ls -lah ${DEVICE:-/srv/node*}/data01/objects/979/c84/f4d1c4252b8349885721271db7bcfc84" # [Handoff]
ssh 10.122.138.242 "ls -lah ${DEVICE:-/srv/node*}/data02/objects/979/c84/f4d1c4252b8349885721271db7bcfc84" # [Handoff]
ssh 10.122.138.243 "ls -lah ${DEVICE:-/srv/node*}/data02/objects/979/c84/f4d1c4252b8349885721271db7bcfc84" # [Handoff]
ssh 10.122.138.241 "ls -lah ${DEVICE:-/srv/node*}/data02/objects/979/c84/f4d1c4252b8349885721271db7bcfc84" # [Handoff]

note: `/srv/node*` is used as default value of `devices`, the real value is set in the config file on each storage node.
[root@swift01 ~]# 
```


