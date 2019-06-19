---
 title: 实践-对象存储Swift使用存储策略
 date:  2019/06/19
 tags: 
   分布式存储
---

## 环境
|  OS  |  Kernel | IP     | Version| Role | Note |
|:-----|:--------|:-----|:---------|:-----|:-----|
|CentOS 7.6|3.10.0|10.122.138.241|openstack ocata| proxy/storage |-|
|CentOS 7.6|3.10.0|10.122.138.242|openstack ocata| proxy/storage |-|
|CentOS 7.6|3.10.0|10.122.138.243|openstack ocata|storage |-|


## 配置存储策略

 
* 编辑swift.conf配置存储策略名称

```bash
[root@swift02 ~]# cat /etc/swift/swift.conf 
[swift-hash]
swift_hash_path_suffix = %SWIFT_HASH_PATH_SUFFIX%

[storage-policy:0]    //对应的RING文件也是类似*-0.gz
name = copy2          //简称，表示该策略存储使用2个副本
default = yes
 
[storage-policy:1]    //对应的RING文件也是类似*-1.gz
name = copy3          //简称，表示该策略存储使用3个副本

NOTE: 编辑好这个文件请务必分发到集群所有节点
``` 

* 针对Policy-1配置生成RING环文件

```bash
[root@swift02 ~]# swift-ring-builder account-1.builder   create 10 3 1    //表示存储3个副本
[root@swift02 ~]# swift-ring-builder container-1.builder create 10 3 1    //表示存储3个副本
[root@swift02 ~]# swift-ring-builder object-1.builder    create 10 3 1    //表示存储3个副本

[root@swift02 ~]# swift-ring-builder account-1.builder add --region 1 --zone 1 --ip 10.122.138.241 --port 6002 --device data01 --weight 100
[root@swift02 ~]# swift-ring-builder account-1.builder add --region 1 --zone 1 --ip 10.122.138.241 --port 6002 --device data02 --weight 100
[root@swift02 ~]# swift-ring-builder account-1.builder add --region 1 --zone 2 --ip 10.122.138.242 --port 6002 --device data01 --weight 100
[root@swift02 ~]# swift-ring-builder account-1.builder add --region 1 --zone 2 --ip 10.122.138.242 --port 6002 --device data02 --weight 100
[root@swift02 ~]# swift-ring-builder account-1.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6002 --device data01 --weight 100
[root@swift02 ~]# swift-ring-builder account-1.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6002 --device data02 --weight 100

[root@swift02 ~]# swift-ring-builder container-1.builder add --region 1 --zone 1 --ip 10.122.138.241 --port 6001 --device data01 --weight 100
[root@swift02 ~]# swift-ring-builder container-1.builder add --region 1 --zone 1 --ip 10.122.138.241 --port 6001 --device data02 --weight 100
[root@swift02 ~]# swift-ring-builder container-1.builder add --region 1 --zone 2 --ip 10.122.138.242 --port 6001 --device data01 --weight 100
[root@swift02 ~]# swift-ring-builder container-1.builder add --region 1 --zone 2 --ip 10.122.138.242 --port 6001 --device data02 --weight 100
[root@swift02 ~]# swift-ring-builder container-1.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6001 --device data01 --weight 100
[root@swift02 ~]# swift-ring-builder container-1.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6001 --device data02 --weight 100

[root@swift02 ~]# swift-ring-builder object-1.builder add --region 1 --zone 1 --ip 10.122.138.241 --port 6000 --device data01 --weight 100 
[root@swift02 ~]# swift-ring-builder object-1.builder add --region 1 --zone 1 --ip 10.122.138.241 --port 6000 --device data02 --weight 100 
[root@swift02 ~]# swift-ring-builder object-1.builder add --region 1 --zone 2 --ip 10.122.138.242 --port 6000 --device data01 --weight 100 
[root@swift02 ~]# swift-ring-builder object-1.builder add --region 1 --zone 2 --ip 10.122.138.242 --port 6000 --device data02 --weight 100 
[root@swift02 ~]# swift-ring-builder object-1.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6000 --device data01 --weight 100 
[root@swift02 ~]# swift-ring-builder object-1.builder add --region 1 --zone 3 --ip 10.122.138.243 --port 6000 --device data02 --weight 100 

[root@swift02 ~]# swift-ring-builder account-1.builde     rebalance
[root@swift02 ~]# swift-ring-builder container-1.builder  rebalance
[root@swift02 ~]# swift-ring-builder object-1.builder     rebalance

NOTE: 配置好RING环之后请务必把生成的\*.builder、\*.gz文件分发到集群所有节点
```


* 查看Policy-1配置的RING环

```bash
[root@swift02 swift]# swift-ring-builder account-1.builder  
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
[root@swift02 swift]# swift-ring-builder container-1.builder 
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
[root@swift02 swift]# swift-ring-builder object-1.builder 
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

## 验证

* 使用copy3这个存储策略来上传文件

NOTE: 存储策略是绑定在container上的！！！

```bash

[root@swift02 ~]# curl -i -XPUT \
-H "X-Auth-Token: LAUGHING_tk74bcf845cbe54e88933b893377486c1c" \
-H "X-Storage-Policy: copy3" \
http://10.122.138.241:8080/v1/LAUGHING_admin/testcopy3con     //创建一个容器指定使用存储策略copy3

[root@swift02 ~]# dd if=/dev/zero of=66M.zip bs=66M count=1
[root@swift02 ~]# curl -i -XPUT -T 66M.zip \
-H "X-Storage-Policy: copy3" \          //这里不指定存储策略也可以，因为testcopy3con容器的默认存储策略就是copy3
-H "X-Auth-Token: LAUGHING_tk74bcf845cbe54e88933b893377486c1c" \
http://10.122.138.241:8080/v1/LAUGHING_admin/testcopy3con/66M.zip   //上传本地的66M.zip文件到testcopy3con这个容器中
```


* 验证文件是否成功应用了copy3这个存储策略

```bash
[root@swift02 ~]# curl -i http://10.122.138.241:8080/endpoints/LAUGHING_admin/testcopy3con/66M.zip
HTTP/1.1 200 OK
Content-Length: 231
Content-Type: application/json
X-Trans-Id: tx75068504ded74e3080556-005d09d87b
X-Openstack-Request-Id: tx75068504ded74e3080556-005d09d87b
X-Trans-Id: tx06ecbcf6a1e64acf89769-005d09d87b
X-Openstack-Request-Id: tx06ecbcf6a1e64acf89769-005d09d87b
Date: Wed, 19 Jun 2019 06:38:51 GMT

["http://10.122.138.242:6000/data02/753/LAUGHING_admin/testcopy3con/66M.zip", "http://10.122.138.243:6000/data01/753/LAUGHING_admin/testcopy3con/66M.zip", "http://10.122.138.241:6000/data01/753/LAUGHING_admin/testcopy3con/66M.zip"][root@swift02 ~]#
```