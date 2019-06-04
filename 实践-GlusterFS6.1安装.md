---
  title: 实践-GlusterFS6.1安装
  date: 2019/06/04
  tags: 
     glusterfs
---

## 环境
| OS      |  Kernel |IP   |Hosts|
|:----    |:--------|:----|:----|
|CentOS7.6|3.10.0   |10.99.72.4|gfs01
|CentOS7.6|3.10.0   |10.99.72.5|gfs02
|CentOS7.6|3.10.0   |10.99.72.6|gfs03

## 先决条件
* 配置机器的hosts（所有节点执行）

```bash
cat /etc/hosts

10.99.72.4 gfs01
10.99.72.5 gfs02
10.99.72.6 gfs03
```

* 清空防护墙规则或关闭（所有节点执行）

```bash
iptables -F 
iptables -X 
iptables -P INPUT ACCEPT
```


* 准备好一块磁盘（这里使用本地回环设备模拟，所有节点执行）

```bash
dd if=/dev/zero of=20G.img count=20 bs=1G   //准备磁盘
mkfs.xfs 20G.img                            //格式化磁盘，xfs
mkdir -p /data/gluster_storage/data01/      //创建挂载点
mount -o loop 20G.img  /data/gluster_storage/data01/  //挂载磁盘 

```


* 配置GlusetFS的yum仓库(所有节点执行)

```bash
yum install centos-release-gluster
```

## 安装
* 安装(所有节点执行)

```bash
yum install glusterfs-server //截至目前安装的是glusterfs-server6.1版本
```

* 查看安装

```bash
[root@xxxxxxxx ~]# rpm -qa |grep gluster
glusterfs-api-6.1-1.el7.x86_64
glusterfs-server-6.1-1.el7.x86_64
centos-release-gluster6-1.0-1.el7.centos.noarch
glusterfs-libs-6.1-1.el7.x86_64
glusterfs-6.1-1.el7.x86_64
glusterfs-cli-6.1-1.el7.x86_64
glusterfs-client-xlators-6.1-1.el7.x86_64
glusterfs-fuse-6.1-1.el7.x86_64
```

* 启动glusterd服务(所有节点执行)

```bash
systemctl  start glusterd
```

## 初始化集群
* 配置信任池(任意节点执行)

```bash
[root@xxxxxxxx ~]# gluster peer probe gfs02
peer probe: success.
[root@xxxxxxxx ~]# gluster peer probe gfs03
peer probe: success.

[root@xxxxxxxx ~]# gluster peer  status
Number of Peers: 2

Hostname: gfs02
Uuid: 904e7d31-2d2e-482d-87ae-4341442d17db
State: Peer in Cluster (Connected)

Hostname: gfs03
Uuid: e9965dee-5e3c-40bb-8bd2-32fb61df7270
State: Peer in Cluster (Connected)
```

## 配置卷

* 创建一个卷目录（所有节点执行)

```bash
mkdir /data/gluster_storage/data01/gv0
```

* 创建一个副本卷，3个副本（任意节点执行）

```bash
[root@xxxxxxxx ~]# gluster volume create gv0 replica 3 gfs01:/data/gluster_storage/data01/gv0 gfs02:/data/gluster_storage/data01/gv0 gfs03:/data/gluster_storage/data01/gv0
volume create: gv0: success: please start the volume to access data
```

* 启动这个副本卷并查看 （任意节点执行）

```bash
[root@xxxxxxxx ~]# gluster volum start gv0
volume start: gv0: success
[root@xxxxxxxx ~]# gluster volum info

Volume Name: gv0
Type: Replicate
Volume ID: 59de7f28-45e5-4576-a4e5-125412e6ee80
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: gfs01:/data/gluster_storage/data01/gv0
Brick2: gfs02:/data/gluster_storage/data01/gv0
Brick3: gfs03:/data/gluster_storage/data01/gv0
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

## 客户端挂载
* 由于我们没有额外的机器，并且如需挂载也需要安装额外的客户端软件包，我们这里使用gfs03这台机器充当客户端来尝试挂载

```bash
[root@xxxxxxxx ~]# mount -t glusterfs  gfs01:/gv0 /mnt 
[root@xxxxxxxx ~]# mount |grep mnt
gfs01:/gv0 on /mnt type fuse.glusterfs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072) //比表示挂载成功
```

* 写入/读取测试

```bash
[root@xxxxxxxx mnt]# echo "hello,world" > test.txt
[root@xxxxxxxx mnt]# ll
total 1
-rw-r--r-- 1 root root 12 Jun  4 11:27 test.txt
[root@xxxxxxxx mnt]# cat test.txt
hello,world
```

* 在glusterfs服务端数据目录验证文件是否存在（所有节点都执行验证）

```bash
[root@ xxxxxxxx mnt]# ll /data/gluster_storage/data01/gv0/
total 8
-rw-r--r-- 2 root root 12 Jun  4 11:27 test.txt
//文件存在，说明副本卷配置正确
```

