
---
title: 实践-DockerCE安装
date: 2019/05/29
tags: 
   docker
---
### 环境
|Os          |Kernel                     |
|------------|---------------------------|
|CentOS 7.6  |3.10.0-957.12.2.el7.x86_64 |


### 卸载老的Docker引擎

```bash
yum remove docker \
        docker-client \
        docker-client-latest \
        docker-common \
        docker-latest \
        docker-latest-logrotate \
        docker-logrotate \
        docker-engine
```

### 安装依赖的包和YUM管理工具

```bash
 yum install -y yum-utils device-mapper-persistent-data lvm2
```
#### 设置Docker的YUM仓库

```bash
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

### 启用Docker仓库

```bash
 yum-config-manager --enable docker-ce-nightly
```

### 查看Docker可选版本

```bash
yum list docker-ce --showduplicates | sort -r
```

#### 安装Docker-CE最新版本（默认安装最新版）

```bash
yum install docker-ce docker-ce-cli containerd.io
```

### 启动Docker服务

```bash
systemctl start docker
```
### 设置开机启动Docker

```bash
systemctl enable docker
```


