---
  title: 实践-使用客户端挂载GlusterFS6.1卷
  date: 2019/06/05
  tags: 
     glusterfs
---

## 服务端环境
| OS      |  Kernel |IP   |Role|
|:----    |:--------|:----|:----|
|CentOS7.6|3.10.0   |10.99.72.4|gfs01
|CentOS7.6|3.10.0   |10.99.72.5|gfs02
|CentOS7.6|3.10.0   |10.99.72.6|gfs03

## 先决条件
* 配置客户端机器的hosts（所有节点执行）

```bash
cat /etc/hosts

10.99.72.4 gfs01
10.99.72.5 gfs02
10.99.72.6 gfs03
```

* 配置GlusetFS的yum仓库

```bash
yum install centos-release-gluster
```

* 加载fuse内核模块

```bash
modprobe fuse
lsmod |grep fuse //确认是否加载成功
```

* 安装Gluster-fuse包

```bash
yum install glusterfs-fuse
```

## 开始挂载
* 挂载

```bash
mkdir /testdir                          //创建挂载点
mount -t glusterfs gfs01:/gv0 /testdir  //上面不配置hosts这里挂载失败
```

* 确认挂载

```bash
df -Th |grep testdir   //服务端的pv0卷需开启df，这里已经开启否则看不到
mount |grep gv0    //确认是否成功挂载
```


* 挂载失败排错

```bash
cat -n /var/log/{YOUR_MOUNT_POINT}.log  //查看日志，对症分析
```
