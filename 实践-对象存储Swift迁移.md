---
 title: 实践-对象存储Swift迁移
 date:  2019/06/14
 tags: 
   分布式存储
---

## 环境
|  OS  |  Kernel | IP     | Version| Role | Note |
|:-----|:--------|:-----|:---------|:-----|:-----|
|CentOS 7.6|3.10.0|10.122.138.231|openstack ocata| proxy/storage |迁移前|
|CentOS 7.6|3.10.0|10.122.138.232|openstack ocata| proxy/storage |迁移前|
|CentOS 7.6|3.10.0|**10.122.138.241**|openstack ocata| proxy/storage |迁移后|
|CentOS 7.6|3.10.0|**10.122.138.242**|openstack ocata| proxy/storage |迁移后|

## 先决条件

**NOTE: 以下所有操作在所有节点执行**

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


* 确保以下hosts文件（/etc/hosts）

```bash
10.122.138.231 swift01
10.122.138.232 swift02
```
 


## 迁移

* 停止swift及相关依赖服务

```bash
[root@swift01 ~]# sh swift.sh proxy_stop
[root@swift01 ~]# sh swift.sh stop
[root@swift01 ~]# systemctl stop memcached 
[root@swift01 ~]# systemctl stop rsyncd 
```

* 查看**迁移前**集群RING哈希环

```bash
[root@swift01 swift]# swift-ring-builder account.builder 
account.builder, build version 5
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 4 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file account.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.231:6002 10.122.138.231:6002 data01 100.00        512    0.00       
            1      1    1 10.122.138.231:6002 10.122.138.231:6002 data02 100.00        512    0.00       
            2      1    2 10.122.138.232:6002 10.122.138.232:6002 data01 100.00        512    0.00       
            3      1    2 10.122.138.232:6002 10.122.138.232:6002 data02 100.00        512    0.00       
[root@swift01 swift]# 
[root@swift01 swift]# swift-ring-builder container.builder 
container.builder, build version 5
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 4 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file container.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.231:6001 10.122.138.231:6001 data01 100.00        512    0.00       
            1      1    1 10.122.138.231:6001 10.122.138.231:6001 data02 100.00        512    0.00       
            2      1    2 10.122.138.232:6001 10.122.138.232:6001 data01 100.00        512    0.00       
            3      1    2 10.122.138.232:6001 10.122.138.232:6001 data02 100.00        512    0.00       
[root@swift01 swift]# 
[root@swift01 swift]# swift-ring-builder object.builder 
object.builder, build version 5
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 4 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file object.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.231:6000 10.122.138.231:6000 data01 100.00        512    0.00       
            1      1    1 10.122.138.231:6000 10.122.138.231:6000 data02 100.00        512    0.00       
            2      1    2 10.122.138.232:6000 10.122.138.232:6000 data01 100.00        512    0.00       
            3      1    2 10.122.138.232:6000 10.122.138.232:6000 data02 100.00        512    0.00       
[root@swift01 swift]# 
``` 
 


* 配置迁移,修改哈希环配置

**NOTE: 以下操作在任意节点配置，之后把最新的RING文件account.builder/container.builder/object.builder等gz文件分发到全部节点**

```bash

//account.builder 
[root@swift01 swift]# swift-ring-builder account.builder set_info --ip 10.122.138.231 --change-ip 10.122.138.241 --change-replication-ip 10.122.138.241       //更换IP地址和复制IP
Matched more than one device:
    d0r1z1-10.122.138.231:6002R10.122.138.231:6002/data01_""
    d1r1z1-10.122.138.231:6002R10.122.138.231:6002/data02_""
Are you sure you want to update the info for these 2 devices? (y/N) y
Device d0r1z1-10.122.138.231:6002R10.122.138.231:6002/data01_"" is now d0r1z1-10.122.138.241:6002R10.122.138.241:6002/data01_""
Device d1r1z1-10.122.138.231:6002R10.122.138.231:6002/data02_"" is now d1r1z1-10.122.138.241:6002R10.122.138.241:6002/data02_""
[root@swift01 swift]# 
[root@swift01 swift]# swift-ring-builder account.builder set_info --ip 10.122.138.232 --change-ip 10.122.138.242 --change-replication-ip 10.122.138.242 
Matched more than one device:
    d2r1z2-10.122.138.232:6002R10.122.138.232:6002/data01_""
    d3r1z2-10.122.138.232:6002R10.122.138.232:6002/data02_""
Are you sure you want to update the info for these 2 devices? (y/N) y
Device d2r1z2-10.122.138.232:6002R10.122.138.232:6002/data01_"" is now d2r1z2-10.122.138.242:6002R10.122.138.242:6002/data01_""
Device d3r1z2-10.122.138.232:6002R10.122.138.232:6002/data02_"" is now d3r1z2-10.122.138.242:6002R10.122.138.242:6002/data02_""
[root@swift01 swift]# 

//container.builder 
[root@swift01 swift]# swift-ring-builder container.builder  set_info --ip 10.122.138.231 --change-ip 10.122.138.241 --change-replication-ip 10.122.138.241        //更换IP地址和复制IP
Matched more than one device:
    d0r1z1-10.122.138.231:6001R10.122.138.231:6001/data01_""
    d1r1z1-10.122.138.231:6001R10.122.138.231:6001/data02_""
Are you sure you want to update the info for these 2 devices? (y/N) y
Device d0r1z1-10.122.138.231:6001R10.122.138.231:6001/data01_"" is now d0r1z1-10.122.138.241:6001R10.122.138.241:6001/data01_""
Device d1r1z1-10.122.138.231:6001R10.122.138.231:6001/data02_"" is now d1r1z1-10.122.138.241:6001R10.122.138.241:6001/data02_""
[root@swift01 swift]# 
[root@swift01 swift]# swift-ring-builder container.builder  set_info --ip 10.122.138.232 --change-ip 10.122.138.242 --change-replication-ip 10.122.138.242
Matched more than one device:
    d2r1z2-10.122.138.232:6001R10.122.138.232:6001/data01_""
    d3r1z2-10.122.138.232:6001R10.122.138.232:6001/data02_""
Are you sure you want to update the info for these 2 devices? (y/N) y
Device d2r1z2-10.122.138.232:6001R10.122.138.232:6001/data01_"" is now d2r1z2-10.122.138.242:6001R10.122.138.242:6001/data01_""
Device d3r1z2-10.122.138.232:6001R10.122.138.232:6001/data02_"" is now d3r1z2-10.122.138.242:6001R10.122.138.242:6001/data02_""
[root@swift01 swift]# 

//object.builder 
[root@swift01 swift]# swift-ring-builder object.builder  set_info --ip 10.122.138.231 --change-ip 10.122.138.241 --change-replication-ip 10.122.138.241          //更换IP地址和复制IP
Matched more than one device:
    d0r1z1-10.122.138.231:6000R10.122.138.231:6000/data01_""
    d1r1z1-10.122.138.231:6000R10.122.138.231:6000/data02_""
Are you sure you want to update the info for these 2 devices? (y/N) y
Device d0r1z1-10.122.138.231:6000R10.122.138.231:6000/data01_"" is now d0r1z1-10.122.138.241:6000R10.122.138.241:6000/data01_""
Device d1r1z1-10.122.138.231:6000R10.122.138.231:6000/data02_"" is now d1r1z1-10.122.138.241:6000R10.122.138.241:6000/data02_""
[root@swift01 swift]# 
[root@swift01 swift]# swift-ring-builder object.builder  set_info --ip 10.122.138.232 --change-ip 10.122.138.242 --change-replication-ip 10.122.138.242  
Matched more than one device:
    d2r1z2-10.122.138.232:6000R10.122.138.232:6000/data01_""
    d3r1z2-10.122.138.232:6000R10.122.138.232:6000/data02_""
Are you sure you want to update the info for these 2 devices? (y/N) y
Device d2r1z2-10.122.138.232:6000R10.122.138.232:6000/data01_"" is now d2r1z2-10.122.138.242:6000R10.122.138.242:6000/data01_""
Device d3r1z2-10.122.138.232:6000R10.122.138.232:6000/data02_"" is now d3r1z2-10.122.138.242:6000R10.122.138.242:6000/data02_""
[root@swift01 swift]# 
[root@swift01 swift]# swift-ring-builder account.builder    write_ring         //写入修改到RING环
[root@swift01 swift]# swift-ring-builder container.builder  write_ring         //写入修改到RING环
[root@swift01 swift]# swift-ring-builder object.builder     write_ring         //写入修改到RING环
```

* 分发哈希环RING文件到集群所有节点

```bash
[root@swift02 swift]# scp -r -P 22 root@10.122.138.231:/etc/swift/*.gz      /etc/swift/
[root@swift02 swift]# scp -r -P 22 root@10.122.138.231:/etc/swift/*.builer  /etc/swift/
```

* 查看最新的swift哈希环

```bash
[root@swift01 swift]# swift-ring-builder account.builder  
account.builder, build version 5
1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 4 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file account.ring.gz is obsolete
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
Ring file container.ring.gz is obsolete
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
Ring file object.ring.gz is obsolete
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6000 10.122.138.241:6000 data01 100.00        512    0.00       
            1      1    1 10.122.138.241:6000 10.122.138.241:6000 data02 100.00        512    0.00       
            2      1    2 10.122.138.242:6000 10.122.138.242:6000 data01 100.00        512    0.00       
            3      1    2 10.122.138.242:6000 10.122.138.242:6000 data02 100.00        512    0.00       
[root@swift01 swift]# 
```

* 人工模拟迁移后的IP地址变更

```bash
更换IP地址过程略,请自行配置，网卡配置文件在-/etc/sysconfig/network-scripts/ifcfg-*
```

* 确保swift服务、memcahced服务、rsyncd服务等配置文件中新IP地址的正确

```bash
 swift:
   - /etc/swift/account-server.conf 
   - /etc/swift/container-server.conf
   - /etc/swift/object-server.conf
   - /etc/swift/proxy-server.conf 
   
 rsyncd:
   - /etc/rsyncd.conf 

 memcached: 
   - /etc/sysconfig/memcached    
```


## 启动服务

* 启动memcached

```bash
[root@swift01 swift]# systemctl start memcached 
```

* 启动rsyncd

```bash
[root@swift01 swift]# systemctl start rsyncd 
```

* 启动swift

```bash
[root@swift01 ~]# sh swift.sh proxy_start
[root@swift01 ~]# sh swift.sh start
```

## 验证

```bash
[root@swift01 ~]# swift -A http://10.122.138.241:8080/auth/v1.0 -U admin:admin -K admin stat -vv
                     StorageURL: http://10.122.138.241:8080/v1/LAUGHING_admin
                     Auth Token: LAUGHING_tkf41e5518d22942019efdd4411465f604
                        Account: LAUGHING_admin
                     Containers: 1
                        Objects: 2
                          Bytes: 335544320
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 2
     Bytes in policy "policy-0": 335544320
         X-Openstack-Request-Id: txb7a88b881b524f39a03a2-005d036756, txeea5db4ef09b435797c31-005d036756, txfde77a7758024f878721e-005d036756, tx1f58de59732d45f2ac740-005d036756
                    X-Timestamp: 1560334114.04390
                     X-Trans-Id: txb7a88b881b524f39a03a2-005d036756, txeea5db4ef09b435797c31-005d036756, txfde77a7758024f878721e-005d036756, tx1f58de59732d45f2ac740-005d036756
                   Content-Type: text/plain; charset=utf-8
                  Accept-Ranges: bytes
[root@swift01 ~]# 
```
