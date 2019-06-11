---
 title: 实践-Docker配置私有仓库安全访问
 date:  2019/06/11 
 tags: 
   docker
---

## 环境
|  OS  |  Kernel | DockerVersion| RegistryDomain|  Registry  |
|:-----|:--------|:-------------|:--------------|:-----------|
|CentOS 7.6|3.10.0|18.06.1-ce|registry.leocloud.club|Harbor|

## 先决条件

* 请确保你有一个可以安全访问(HTTPS访问)的Docker Registry仓库
* 确保hosts文件正确

```bash
10.99.73.32 registry.leocloud.club
NOTE: 我这里已经提前配置好了Docker镜像仓库，使用开源的Harbor搭建
```

## Docker配置

* 配置证书

```bash
[root@qs-storage-swift-03 registry.leocloud.club]# pwd
/etc/docker/certs.d/registry.leocloud.club
[root@qs-storage-swift-03 registry.leocloud.club]# ll
total 12
-rw-r--r-- 1 root root 2094 Jun 11 10:46 ca.crt
-rw-r--r-- 1 root root 2009 Jun 11 10:46 registry.leocloud.club.cert //必须以为cert结尾
-rw-r--r-- 1 root root 3272 Jun 11 10:46 registry.leocloud.club.key
[root@qs-storage-swift-03 registry.leocloud.club]#
```

* 重新启动Docker服务

```bash
[root@qs-storage-swift-03 ~]# systemctl  restart docker
```

## 配置系统信任私有CA（如果证书是非自签发证书可忽略这步）

* 添加私有CA证书文件到系统信任目录

```bash
[root@qs-storage-swift-03 ~]# cp ca.crt /etc/pki/ca-trust/source/anchors/
```

* 更新系统CA证书库

```bash
[root@qs-storage-swift-03 ~]# yum install -y ca-certificates 
[root@qs-storage-swift-03 ~]# update-ca-trust
```


## 验证
* 登录到安全的仓库

```bash
[root@qs-storage-swift-03 ~]# docker login registry.leocloud.club
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

