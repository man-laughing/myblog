---
 title: 实践-对象存储OpenStackSwift安装
 date:  2019/06/12
 tags: 
   分布式存储
---

## 环境
|  OS  |  Kernel | IP     | Version| Role |
|:-----|:--------|:-----|:---------|:-----|
|CentOS 7.6|3.10.0|10.122.138.231|openstack ocata| proxy/storage |
|CentOS 7.6|3.10.0|10.122.138.232|openstack ocata| proxy/storage |

## 先决条件

#### NOTE：以下所有操作在所有节点执行

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
* 配置OpenStack的YUM仓库


```bash
[root@localhost ~]#  yum install -y centos-release-openstack-ocata
```

* 准备2块硬盘作为存储盘（这里使用vmwar虚拟机单独添加了2块10GB的磁盘）且格式化为xfs文件系统

```bash

//使用fdisk分区过程略
[root@swift01 ~]# mkfs.xfs /dev/sdb1
[root@swift01 ~]# mkfs.xfs /dev/sdc1
```

* 挂载磁盘

```bash
[root@swift01 ~]# mount /dev/sdb1  /data/swiftstorage/data01
[root@swift01 ~]# mount /dev/sdc1  /data/swiftstorage/data02
[root@swift01 ~]# df -THl
文件系统                类型      容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root xfs        19G  1.3G   18G    7% /
devtmpfs                devtmpfs  234M     0  234M    0% /dev
tmpfs                   tmpfs     246M     0  246M    0% /dev/shm
tmpfs                   tmpfs     246M  5.8M  240M    3% /run
tmpfs                   tmpfs     246M     0  246M    0% /sys/fs/cgroup
/dev/sda1               xfs       1.1G  139M  925M   14% /boot
tmpfs                   tmpfs      50M     0   50M    0% /run/user/0
/dev/sdb1               xfs        11G   34M   11G    1% /data/swiftstorage/data01
/dev/sdc1               xfs        11G   34M   11G    1% /data/swiftstorage/data02
```

## 配置memcached服务

#### NOTE：以下所有操作在所有节点执行

* 安装memcached

```bash
[root@swift02 ~]# yum install -y memcached 
```

* 启动memcahed服务

```bash
[root@swift02 ~]# systemctl start memcahed 
```

## 配置rsync服务

#### NOTE：以下所有操作在所有节点执行
* 安装rsync

```bash
[root@swift02 ~]# yum install -y rsync
```

* 配置rsync(/etc/rsyncd.conf)

```bash
uid = swift   //swift文件复制依赖rsync，这里使用swift运行用户swift保证权限无碍
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = 10.122.138.231        //填写本机IP地址，表示rsyncd监听的地址

[account]
max connections = 1024
path = /data/swiftstorage/      //这里为swift存储根目录
read only = False
lock file = /var/lock/account.lock
exclude = ".*"

[container]
max connections = 1024
path = /data/swiftstorage/      //这里为swift存储根目录
read only = False
lock file = /var/lock/container.lock
exclude = ".*"

[object]
max connections = 1024 
path = /data/swiftstorage/      //这里为swift存储根目录
read only = False
lock file = /var/lock/object.lock
exclude = ".*"
```

* 启动rsync服务

```bash
[root@swift02 ~]# systemctl  start rsyncd
```

## 配置swift服务

#### NOTE：以下所有操作在所有节点执行

* 安装软件包

```bash
[root@swift01 ~]# yum install -y openstack-swift-account openstack-swift-proxy  openstack-swift-container openstack-swift-object python-swift  python2-swiftclient 
```

* 配置account账户服务(/etc/swift/account-server.conf)

```bash
[DEFAULT]
bind_ip = 10.122.138.231       //监听的本机IP，需要和ring环的匹配
bind_port = 6002               //监听的本机端口，需要和ring环的匹配
user = swift
swift_dir = /etc/swift
devices = /data/swiftstorage/  //填写存储的根目录，需要和ring环的匹配
mount_check = True
workers = 2
max_clients = 65535
log_name = account
log_facility = LOG_LOCAL0
log_level = INFO
[pipeline:main]
pipeline = healthcheck recon account-server
[app:account-server]
use = egg:swift#account
set log_name = account
set log_facility = LOG_LOCAL2
set log_level = INFO
[filter:healthcheck]
use = egg:swift#healthcheck
[filter:recon]
use = egg:swift#recon
recon_cache_path = /var/cache/swift
[account-replicator]
[account-auditor]
[account-reaper]
[filter:xprofile]
use = egg:swift#xprofile
```

* 配置container容器服务(/etc/swift/account-server.conf)

```bash
[DEFAULT]
bind_ip = 10.122.138.231          //监听的本机IP，需要和ring环的匹配
bind_port = 6001                  //监听的本机端口，需要和ring环的匹配
user = swift
swift_dir = /etc/swift
devices = /data/swiftstorage/     //填写存储的根目录，需要和ring环的匹配
mount_check = True
workers = 2
max_clients = 65535
log_name = container
log_facility = LOG_LOCAL0
log_level = INFO
[pipeline:main]
pipeline = healthcheck recon container-server
[app:container-server]
use = egg:swift#container
set log_name = container
set log_facility = LOG_LOCAL3
set log_level = INFO
[filter:healthcheck]
use = egg:swift#healthcheck
[filter:recon]
use = egg:swift#recon
[container-replicator]
[container-updater]
[container-auditor]
[container-sync]
[filter:xprofile]
use = egg:swift#xprofile
```

* 配置object对象服务(/etc/swift/account-server.conf)

```bash
[DEFAULT]
bind_ip = 10.122.138.231            //监听的本机IP，需要和ring环的匹配
bind_port = 6000                    //监听的本机端口，需要和ring环的匹配
user = swift
swift_dir = /etc/swift
devices = /data/swiftstorage/       //填写存储的根目录，需要和ring环的匹配
mount_check = True
workers = 2
backlog = 65535
max_clients = 65535
log_name = object
log_facility = LOG_LOCAL0
log_level = INFO
[pipeline:main]
pipeline = healthcheck recon object-server
[app:object-server]
use = egg:swift#object
set log_name = object
set log_facility = LOG_LOCAL4
set log_level = INFO
threads_per_disk = 4
[filter:healthcheck]
use = egg:swift#healthcheck
[filter:recon]
use = egg:swift#recon
recon_cache_path = /var/cache/swift
recon_lock_path = /var/lock
[object-replicator]
daemonize = on
concurrency = 4
rsync_bwlimit = 0
http_timeout = 180
[object-reconstructor]
daemonize = on
concurrency = 4
[object-updater]
[object-auditor]
concurrency = 4
[filter:xprofile]
use = egg:swift#xprofile
```

* 配置proxy服务(/etc/swift/proxy-server.conf)

```bash
[DEFAULT]
bind_ip = 10.122.138.231
bind_port = 8080
user = swift
workers = 2

[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck cache list-endpoints tempurl tempauth proxy-logging dlo versioned_writes proxy-server   //这是proxy处理请求的管道流程

[filter:catch_errors]
use = egg:swift#catch_errors

[filter:gatekeeper]
use = egg:swift#catch_errors

[filter:healthcheck]
use = egg:swift#healthcheck

[filter:cache]
use = egg:swift#memcache
memcache_servers = 10.122.138.231:11211,10.122.138.232:11211   //填写多个memcached地址组成集群使用

[filter:list-endpoints]
use = egg:swift#list_endpoints

[filter:tempurl]                    
use = egg:swift#tempurl

[filter:tempauth]                                       //表示临时认证服务
use = egg:swift#tempauth
user_admin_admin = admin .admin .reseller_admin         //临时认证的账户密码信息
user_test_tester = testing .admin
reseller_prefix = LAUGHING
token_life = 86400                                      //通过验证会获得一个token，这个选项表示了token的生存时间，单位是秒

[filter:proxy-logging]
use = egg:swift#proxy_logging

[filter:dlo]
use = egg:swift#catch_errors

[filter:versioned_writes]
use = egg:swift#catch_errors

[app:proxy-server]
use = egg:swift#proxy
allow_account_management = true
account_autocreate = true
set log_name = proxy
set log_facility = LOG_LOCAL1
set log_level = INFO
log_headers = false

[filter:bulk]
use = egg:swift#bulk
```


* 配置ring环

###### NOTE: 以下配置在任意节点执行就好，然后把生成的文件copy到其他节点

```bash
//配置account账户哈希环
[root@swift01 ~]# cd /etc/swift/
[root@swift01 swift]# swift-ring-builder account.builder create 10 2 1      //表示2^10个分区，2个副本，1个区域
[root@swift01 swift]# swift-ring-builder account.builder add --region 1 --zone 1 --ip 10.122.138.231 --port 6002 --device data01 --weight 100 
[root@swift01 swift]# swift-ring-builder account.builder add --region 1 --zone 1 --ip 10.122.138.231 --port 6002 --device data02 --weight 100 
[root@swift01 swift]# swift-ring-builder account.builder add --region 1 --zone 2 --ip 10.122.138.232 --port 6002 --device data01 --weight 100 
[root@swift01 swift]# swift-ring-builder account.builder add --region 1 --zone 2 --ip 10.122.138.232 --port 6002 --device data02 --weight 100 
[root@swift01 swift]# swift-ring-builder account.builder rebalance          //表示数据分区重新均衡
//配置container容器哈希环
[root@swift01 swift]# swift-ring-builder container.builder create 10 2 1    //表示2^10个分区，2个副本，1个区域
[root@swift01 swift]# swift-ring-builder container.builder add --region 1 --zone 1 --ip 10.122.138.231 --port 6001 --device data01 --weight 100 
[root@swift01 swift]# swift-ring-builder container.builder add --region 1 --zone 1 --ip 10.122.138.231 --port 6001 --device data02 --weight 100 
[root@swift01 swift]# swift-ring-builder container.builder add --region 1 --zone 2 --ip 10.122.138.232 --port 6001 --device data01 --weight 100  
[root@swift01 swift]# swift-ring-builder container.builder add --region 1 --zone 2 --ip 10.122.138.232 --port 6001 --device data02 --weight 100 
[root@swift01 swift]# swift-ring-builder container.builder rebalance        //表示数据分区重新均衡
//配置container对象哈希环
[root@swift01 swift]# 
[root@swift01 swift]# swift-ring-builder object.builder create 10 2 1       //表示2^10个分区，2个副本，1个区域
[root@swift01 swift]# swift-ring-builder object.builder add --region 1 --zone 1 --ip 10.122.138.231 --port 6000 --device data01 --weight 100 
[root@swift01 swift]# swift-ring-builder object.builder add --region 1 --zone 1 --ip 10.122.138.231 --port 6000 --device data02 --weight 100 
[root@swift01 swift]# swift-ring-builder object.builder add --region 1 --zone 2 --ip 10.122.138.232 --port 6000 --device data01 --weight 100 
[root@swift01 swift]# swift-ring-builder object.builder add --region 1 --zone 2 --ip 10.122.138.232 --port 6000 --device data02 --weight 100 
[root@swift01 swift]# swift-ring-builder object.builder rebalance           //表示数据分区重新均衡
```  

* 编辑服务启动脚本

```bash
#!/bin/bash

function start() {
systemctl start openstack-swift-account.service
systemctl start openstack-swift-account-auditor.service
systemctl start openstack-swift-account-reaper.service
systemctl start openstack-swift-account-replicator.service
systemctl start openstack-swift-container.service
systemctl start openstack-swift-container-auditor.service
systemctl start openstack-swift-container-replicator.service
systemctl start openstack-swift-container-updater.service
systemctl start openstack-swift-object.service
systemctl start openstack-swift-object-auditor.service
systemctl start openstack-swift-object-replicator.service
systemctl start openstack-swift-object-updater.service
}

function stop() {
systemctl stop openstack-swift-account.service
systemctl stop openstack-swift-account-auditor.service
systemctl stop openstack-swift-account-reaper.service
systemctl stop openstack-swift-account-replicator.service
systemctl stop openstack-swift-container.service
systemctl stop openstack-swift-container-auditor.service
systemctl stop openstack-swift-container-replicator.service
systemctl stop openstack-swift-container-updater.service
systemctl stop openstack-swift-object.service
systemctl stop openstack-swift-object-auditor.service
systemctl stop openstack-swift-object-replicator.service
systemctl stop openstack-swift-object-updater.service
}

function proxy_start() {
   systemctl start openstack-swift-proxy.service
}

function proxy_stop() {
   systemctl stop openstack-swift-proxy.service
}

case $1 in
    start)
        start
    ;;
    stop)
        stop
    ;;
    proxy_start)
        proxy_start
    ;;
    proxy_stop)
        proxy_stop
    ;;
    *)
     echo 'Error'
     exit  2
esac
```

* 设置一些权限

```bash
[root@swift01 ~]# chmod -R swift:swift /etc/swift/ /data/swiftstorage/
```

* 启动proxy服务

```bash
[root@swift01 ~]# sh swift.sh proxy_start
```

* 启动存储(account/container/object)服务

```bash
[root@swift01 ~]# sh swift.sh  start 
```

* 查看服务是否启动

```bash
[root@swift01 ~]# netstat -anputl |grep LISTEN |grep python
tcp        0      0 10.122.138.231:8080     0.0.0.0:*               LISTEN      9073/python2        
tcp        0      0 10.122.138.231:6000     0.0.0.0:*               LISTEN      8919/python2        
tcp        0      0 10.122.138.231:6001     0.0.0.0:*               LISTEN      8861/python2        
tcp        0      0 10.122.138.231:6002     0.0.0.0:*               LISTEN      8833/python2 
```

## 简单的操作实例

#### NOTE：以下验证操作在任意节点执行

* 查看当前集群状态

```bash
[root@swift01 ~]# swift -A http://10.122.138.232:8080/auth/v1.0 -U admin:admin -K admin stat -vv

            StorageURL: http://10.122.138.232:8080/v1/LAUGHING_admin
            Auth Token: LAUGHING_tka5eb9275228d4debbad6523553ac7e0e
               Account: LAUGHING_admin
            Containers: 0
               Objects: 0
                 Bytes: 0
       X-Put-Timestamp: 1560334080.16564
           X-Timestamp: 1560334080.16564
            X-Trans-Id: tx8f3c73c1d2b84287b25d9-005d00cf00, tx5bf02da63b0b4c18b5f60-005d00cf00, tx370f5fa821854356a6346-005d00cf00, txd24656dd585d44f9a3b6e-005d00cf00
          Content-Type: text/plain; charset=utf-8
X-Openstack-Request-Id: tx8f3c73c1d2b84287b25d9-005d00cf00, tx5bf02da63b0b4c18b5f60-005d00cf00, tx370f5fa821854356a6346-005d00cf00, txd24656dd585d44f9a3b6e-005d00cf00
```

* 新建一个容器

```bash
[root@swift01 ~]# swift -A http://10.122.138.232:8080/auth/v1.0 -U admin:admin -K admin post test01 
```

* 删除一个容器

```bash
[root@swift01 ~]# swift -A http://10.122.138.232:8080/auth/v1.0 -U admin:admin -K admin delete test01 
```

* 上传一个文件

```bash
[root@swift01 ~]# dd if=/dev/zero of=10M.test.zip bs=10M count=1  //创建一个10MB的测试文件
[root@swift01 ~]# ll
总用量 10248
-rw-r--r--  1 root root 10485760 6月  12 18:09 10M.test.zip
-rw-------. 1 root root     1323 6月   6 18:32 anaconda-ks.cfg
-rwxr-xr-x  1 root root     1718 6月  12 16:20 swift.sh
[root@swift01 ~]# swift -A http://10.122.138.232:8080/auth/v1.0 -U admin:admin -K admin post test02
[root@swift01 ~]# swift -A http://10.122.138.232:8080/auth/v1.0 -U admin:admin -K admin upload test02 10M.test.zip  
10M.test.zip
```

* 下载一个文件

```bash
[root@swift01 tmp]# swift -A http://10.122.138.232:8080/auth/v1.0 -U admin:admin -K admin download test02 10M.test.zip
10M.test.zip [auth 0.011s, headers 0.025s, total 0.222s, 49.712 MB/s]
[root@swift01 tmp]# ll 
总用量 10244
-rw-r--r--  1 root root 10485760 6月  12 18:09 10M.test.zip
-rwx------. 1 root root      836 6月   6 18:32 ks-script-zclY0U
drwx------  3 root root       17 6月  10 09:46 systemd-private-211132f9b5da479ba262e921f3aae27c-chronyd.service-9IrFir
drwx------  3 root root       17 6月  12 15:14 systemd-private-3982ce91f58842a793f56c4e61fcda6b-chronyd.service-dLQW8z
drwx------  3 root root       17 6月   6 20:43 systemd-private-8e2568d00f4a4416ae415858903c6c1a-chronyd.service-F4Qa3q
-rw-------. 1 root root        0 6月   6 18:27 yum.log
[root@swift01 tmp]# 
```

* 删除一个文件

```bash
[root@swift01 ~]# swift -A http://10.122.138.232:8080/auth/v1.0 -U admin:admin -K admin delete test02 10M.test.zip
```

