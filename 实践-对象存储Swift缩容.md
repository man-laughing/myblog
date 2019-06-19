---
 title: 实践-对象存储Swift缩容
 date:  2019/06/19
 tags: 
   分布式存储
---

## 环境
|  OS  |  Kernel | IP     | Version| Role | Note|
|:-----|:--------|:-----|:---------|:-----|:----|
|CentOS 7.6|3.10.0|10.122.138.241|openstack ocata| proxy/storage |缩容节点|
|CentOS 7.6|3.10.0|10.122.138.242|openstack ocata| proxy/storage |-|
|CentOS 7.6|3.10.0|10.122.138.243|openstack ocata|storage |-|

## 配置缩容

* 查看当前Policy-1的RING环配置

```bash
[root@swift02 swift]# swift-ring-builder account-1.ring.gz  
Note: using account-1.builder instead of account-1.ring.gz as builder file
account-1.builder, build version 7
1024 partitions, 3.000000 replicas, 1 regions, 3 zones, 6 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file account-1.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6002 10.122.138.241:6002 data01 100.00        512    0.00       
            1      1    1 10.122.138.241:6002 10.122.138.241:6002 data02 100.00        512    0.00       
            2      1    2 10.122.138.242:6001 10.122.138.242:6001 data01 100.00        512    0.00       
            3      1    2 10.122.138.242:6001 10.122.138.242:6001 data02 100.00        512    0.00       
            4      1    3 10.122.138.243:6001 10.122.138.243:6001 data01 100.00        512    0.00       
            5      1    3 10.122.138.243:6001 10.122.138.243:6001 data02 100.00        512    0.00       
[root@swift02 swift]# swift-ring-builder container-1.ring.gz  
Note: using container-1.builder instead of container-1.ring.gz as builder file
container-1.builder, build version 7
1024 partitions, 3.000000 replicas, 1 regions, 3 zones, 6 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file container-1.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6001 10.122.138.241:6001 data01 100.00        512    0.00       
            1      1    1 10.122.138.241:6001 10.122.138.241:6001 data02 100.00        512    0.00       
            2      1    2 10.122.138.242:6001 10.122.138.242:6001 data01 100.00        512    0.00       
            3      1    2 10.122.138.242:6001 10.122.138.242:6001 data02 100.00        512    0.00       
            4      1    3 10.122.138.243:6001 10.122.138.243:6001 data01 100.00        512    0.00       
            5      1    3 10.122.138.243:6001 10.122.138.243:6001 data02 100.00        512    0.00       
[root@swift02 swift]# swift-ring-builder object-1.ring.gz  
Note: using object-1.builder instead of object-1.ring.gz as builder file
object-1.builder, build version 7
1024 partitions, 3.000000 replicas, 1 regions, 3 zones, 6 devices, 0.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file object-1.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            0      1    1 10.122.138.241:6000 10.122.138.241:6000 data01 100.00        512    0.00       
            1      1    1 10.122.138.241:6000 10.122.138.241:6000 data02 100.00        512    0.00       
            2      1    2 10.122.138.242:6000 10.122.138.242:6000 data01 100.00        512    0.00       
            3      1    2 10.122.138.242:6000 10.122.138.242:6000 data02 100.00        512    0.00       
            4      1    3 10.122.138.243:6000 10.122.138.243:6000 data01 100.00        512    0.00       
            5      1    3 10.122.138.243:6000 10.122.138.243:6000 data02 100.00        512    0.00       
[root@swift02 swift]# 
```

* 移除10.122.138.241存储节点,对于Policy-1的ring环配置来说

```bash
[root@swift02 swift]# swift-ring-builder account-1.builder  remove --ip 10.122.138.241 -y 
Matched more than one device:
    d0r1z1-10.122.138.241:6002R10.122.138.241:6002/data01_""
    d1r1z1-10.122.138.241:6002R10.122.138.241:6002/data02_""
d0r1z1-10.122.138.241:6002R10.122.138.241:6002/data01_"" marked for removal and will be removed next rebalance.
d1r1z1-10.122.138.241:6002R10.122.138.241:6002/data02_"" marked for removal and will be removed next rebalance.     
[root@swift02 swift]# swift-ring-builder container-1.builder  remove --ip 10.122.138.241 -y
Matched more than one device:
    d0r1z1-10.122.138.241:6001R10.122.138.241:6001/data01_""
    d1r1z1-10.122.138.241:6001R10.122.138.241:6001/data02_""
d0r1z1-10.122.138.241:6001R10.122.138.241:6001/data01_"" marked for removal and will be removed next rebalance.
d1r1z1-10.122.138.241:6001R10.122.138.241:6001/data02_"" marked for removal and will be removed next rebalance.
[root@swift02 swift]# swift-ring-builder object-1.builder  remove --ip 10.122.138.241 -y
Matched more than one device:
    d0r1z1-10.122.138.241:6000R10.122.138.241:6000/data01_""
    d1r1z1-10.122.138.241:6000R10.122.138.241:6000/data02_""
d0r1z1-10.122.138.241:6000R10.122.138.241:6000/data01_"" marked for removal and will be removed next rebalance.
d1r1z1-10.122.138.241:6000R10.122.138.241:6000/data02_"" marked for removal and will be removed next rebalance.
[root@swift02 swift]# 
[root@swift02 swift]# swift-ring-builder account-1.builder  rebalance 
Reassigned 1024 (100.00%) partitions. Balance is now 0.39.  Dispersion is now 0.00
[root@swift02 swift]# swift-ring-builder container-1.builder  rebalance 
Reassigned 1024 (100.00%) partitions. Balance is now 0.13.  Dispersion is now 0.00
[root@swift02 swift]# swift-ring-builder object-1.builder  rebalance 
Reassigned 1024 (100.00%) partitions. Balance is now 0.13.  Dispersion is now 0.00
[root@swift02 swift]# 
```
 
* 查看当前的Policy-1的ring哈希环配置

```bash
[root@swift02 swift]# swift-ring-builder account-1.builder  
account-1.builder, build version 10
1024 partitions, 3.000000 replicas, 1 regions, 2 zones, 4 devices, 0.39 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:59:00 remaining)
The overload factor is 0.00% (0.000000)
Ring file account-1.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            2      1    2 10.122.138.242:6001 10.122.138.242:6001 data01 100.00        771    0.39       
            3      1    2 10.122.138.242:6001 10.122.138.242:6001 data02 100.00        765   -0.39       
            4      1    3 10.122.138.243:6001 10.122.138.243:6001 data01 100.00        765   -0.39       
            5      1    3 10.122.138.243:6001 10.122.138.243:6001 data02 100.00        771    0.39       

[root@swift02 swift]# 
[root@swift02 swift]# 
[root@swift02 swift]# swift-ring-builder container-1.builder  
container-1.builder, build version 10
1024 partitions, 3.000000 replicas, 1 regions, 2 zones, 4 devices, 0.13 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:58:41 remaining)
The overload factor is 0.00% (0.000000)
Ring file container-1.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            2      1    2 10.122.138.242:6001 10.122.138.242:6001 data01 100.00        769    0.13       
            3      1    2 10.122.138.242:6001 10.122.138.242:6001 data02 100.00        767   -0.13       
            4      1    3 10.122.138.243:6001 10.122.138.243:6001 data01 100.00        767   -0.13       
            5      1    3 10.122.138.243:6001 10.122.138.243:6001 data02 100.00        769    0.13       
[root@swift02 swift]# 
[root@swift02 swift]# swift-ring-builder object-1.builder 
object-1.builder, build version 10
1024 partitions, 3.000000 replicas, 1 regions, 2 zones, 4 devices, 0.13 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1 (0:58:41 remaining)
The overload factor is 0.00% (0.000000)
Ring file object-1.ring.gz is up-to-date
Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            2      1    2 10.122.138.242:6000 10.122.138.242:6000 data01 100.00        769    0.13       
            3      1    2 10.122.138.242:6000 10.122.138.242:6000 data02 100.00        767   -0.13       
            4      1    3 10.122.138.243:6000 10.122.138.243:6000 data01 100.00        769    0.13       
            5      1    3 10.122.138.243:6000 10.122.138.243:6000 data02 100.00        767   -0.13       
[root@swift02 swift]# 
NOTE： 请务必把最新的ring文件和builder文件分发到集群所有节点
``` 

 

## 验证

* 查询之前存在Policy-1的容器里的一个文件的对应实际路径

```bash
[root@swift02 ~]# swift -A http://10.122.138.242:8080/auth/v1.0 -U admin:admin -K admin stat  testcopy3con
               Account: LAUGHING_admin
             Container: testcopy3con
               Objects: 2
                 Bytes: 1142947840
              Read ACL:
             Write ACL:
               Sync To:
              Sync Key:
         Accept-Ranges: bytes
      X-Storage-Policy: copy3
         Last-Modified: Wed, 19 Jun 2019 03:47:31 GMT
           X-Timestamp: 1560915945.98737
            X-Trans-Id: txd9f3150c0770409faa529-005d09f7d7, tx021a6f1241bd49639e929-005d09f7d7, txcb26df3eb9124ff696b58-005d09f7d7, tx0442a5af30524055bc3f5-005d09f7d7
          Content-Type: text/plain; charset=utf-8
X-Openstack-Request-Id: txd9f3150c0770409faa529-005d09f7d7, tx021a6f1241bd49639e929-005d09f7d7, txcb26df3eb9124ff696b58-005d09f7d7, tx0442a5af30524055bc3f5-005d09f7d7
[root@swift02 ~]# 
[root@swift02 ~]# curl  http://10.122.138.242:8080/endpoints/LAUGHING_admin/testcopy3con/1gtest.zip
["http://10.122.138.242:6000/data02/161/LAUGHING_admin/testcopy3con/1gtest.zip", "http://10.122.138.243:6000/data01/161/LAUGHING_admin/testcopy3con/1gtest.zip", "http://10.122.138.242:6000/data01/161/LAUGHING_admin/testcopy3con/1gtest.zip"] //可以看到3份副本全部都分散在了242和243两台主机设备上 
```

 