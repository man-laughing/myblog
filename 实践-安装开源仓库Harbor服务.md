---
  title: 实践-安装开源仓库Harbor服务
  date: 2019/06/10
  tags: 
    harbor
---


## 环境

|  OS  |  Kernel |  IP   | Harbor  |Docker |docker-compose |
|:-----|:--------|:------|:--------|:------|:--------------|
|CentOS 7.6|3.10.0|10.99.73.32|1.8.0|18.09.6|1.24.0|


## 安装方式
* 离线安装，下载所需要的docker镜像
* 通过docker-compose启动Harbor服务


## 先决条件
* Docker 17.03.0+    
* docker-compose 1.18.0+  
* openssl LATEST-VERSION

NOTE：以上依赖我都已经提前安装好了


## 配置 

* 下载Harbor离线安装包且解压缩

```bash
[root@prometheus-shqs-1 ~]# mkdir /usr/local/src/harbor_install
[root@prometheus-shqs-1 ~]# cd /usr/local/src/harbor_install
[root@prometheus-shqs-1 harbor_install]# wget -S "https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.0.tgz"
[root@prometheus-shqs-1 harbor_install]# tar xf harbor-offline-installer-v1.8.0.tgz
[root@prometheus-shqs-1 harbor_install]# cd harbor
[root@prometheus-shqs-1 harbor]# ll
total 543152
-rw-r--r-- 1 root root 556153903 May 16 19:55 harbor.v1.8.0.tar.gz
-rw-r--r-- 1 root root      4841 Jun 10 16:47 harbor.yml
-rwxr-xr-x 1 root root      5088 May 16 19:54 install.sh
-rw-r--r-- 1 root root     11347 May 16 19:54 LICENSE
-rwxr-xr-x 1 root root      1654 May 16 19:54 prepare
```

* 配置harbor.yml

```bash
[root@prometheus-shqs-1 harbor]# egrep -v "(^$|^#)"  harbor.yml
hostname: prometheus-shqs-1
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 88
harbor_admin_password: admin
database:
  # The password for the root user of Harbor DB. Change this before any production use.
  password: root123
data_volume: /data/harbor
clair:
  # The interval of clair updaters, the unit is hour, set to 0 to disable the updaters.
  updaters_interval: 12
  # Config http proxy for Clair, e.g. http://my.proxy.com:3128
  # Clair doesn't need to connect to harbor internal components via http proxy.
  http_proxy:
  https_proxy:
  no_proxy: 127.0.0.1,localhost,core,registry
jobservice:
  # Maximum number of job workers in job service
  max_job_workers: 10
chart:
  # Change the value of absolute_url to enabled can enable absolute url in chart
  absolute_url: disabled
log:
  # options are debug, info, warning, error, fatal
  level: info
  # Log files are rotated log_rotate_count times before being removed. If count is 0, old versions are removed rather than rotated.
  rotate_count: 50
  # Log files are rotated only if they grow bigger than log_rotate_size bytes. If size is followed by k, the size is assumed to be in kilobytes.
  # If the M is used, the size is in megabytes, and if G is used, the size is in gigabytes. So size 100, size 100k, size 100M and size 100G
  # are all valid.
  rotate_size: 200M
  # The directory on your host that store log
  location: /var/log/harbor
_version: 1.8.0
```

* 执行安装脚本

```bash
[root@prometheus-shqs-1 harbor]# ./install.sh
```


## 验证 
![](http://demo1.ess.ejucloud.cn/harbor_install.jpg)